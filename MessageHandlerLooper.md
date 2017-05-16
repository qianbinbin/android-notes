# Android 的消息机制

## 概述

Android 的 UI 控件不是线程安全的，因此规定访问 UI 必须在主线程（即 UI 线程）中进行。ViewRootImpl 的`checkThread()`方法对 UI 操作进行验证，如果在后台线程中访问 UI，程序就会抛出异常。

但主线程中不适合进行耗时操作，因为这样会导致 ANR，所以耗时操作应在后台线程中进行。执行完成后，再使用 Handler 切换到主线程进行 UI 操作。

为什么 UI 控件不设计成线程安全的呢？一是避免访问逻辑变复杂，二是避免效率低下。

Handler 是 Android 消息机制的上层接口，通过它可以将一个任务切换到 Handler 所在的线程中执行。Handler 的运行需要 MessageQueue 和 Looper 支撑。MessageQueue 维护了一个 Message 链表，以队列的逻辑结构向 Handler 和 Looper 提供接口，Looper 中维护了一个 MessageQueue 类型的消息队列。Handler 将 Message 加入队列，Looper 从队列中取出 Message，Looper 以无限循环的方式查找是否有需要处理的 Message。Looper 类中 ThreadLocal<Looper> 类型的静态变量`sThreadLocal`，用于为当前线程保存和获取 Looper（ThreadLocal 用于为当前线程保存变量，此变量只能被当前线程访问，其它线程无法访问）。

Handler 创建时，如果不指定 Looper，默认会采用当前线程的 Looper 来构造消息循环系统：

```
mLooper = Looper.myLooper();
```

线程默认是没有 Looper 的，如果不为线程创建 Looper，使用 Handler 时就会抛出异常，所以后台线程中一般要先调用`Looper.prepare()`为当前线程创建一个 Looper。

主线程，即 ActivityThread，其创建时就会初始化 Looper，因此主线程中默认可以使用 Handler。

Handler 创建完毕后，可使用 post 开头的方法（以下简称 post 方法）将一个 Runnable，也可使用 send 开头的方法（以下简称 send 方法）发送一个 Message，本质都是将一个 Message 加入 Handler 所在线程对应的 Looper 中的消息队列。Looper 发现有需要处理的 Message，就会调用对应 Handler 的`dispatchMessage()`方法来处理。因此 Handler 中的业务逻辑就在创建 Handler 所在的线程中执行。这也是后台线程能通过主线程中的 Handler 来更新 UI 的原因。

## Android 消息机制分析

### ThreadLocal 工作原理

ThreadLocal 是一个线程内部的数据存储类，只有在指定线程中可以获取到存储的数据，其它线程无法获取。

不同线程访问同一个 ThreadLocal 对象，它们对 ThreadLocal 的读写仅限于各自线程的内部，即不同线程中维护一套数据副本，彼此互不干扰。Looper、ActivityThread、ActivityManagerService 中都用到了 ThreadLocal。

在开始采用 OpenJDK 的 Android 7.0 中，ThreadLocal 的原理是使用了一个静态嵌套类 ThreadLocalMap（一个定制化的哈希表）。ThreadLocalMap 以 ThreadLocal 对象为 key，以需要保存的对象为 value，提供存取键值对的接口。实际上 ThreadLocalMap 维护了一个 Entry 类型的数组`table`，这些键值对就保存在 Entry 数组中。ThreadLocal 类结构如下：

```
public class ThreadLocal<T> {
    // ...

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    public void remove() {
        ThreadLocalMap m = getMap(Thread.currentThread());
        if (m != null)
            m.remove(this);
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    // ...

    static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }

        private Entry[] table;

        private Entry getEntry(ThreadLocal key) { /* ... */ }

        private void set(ThreadLocal key, Object value) { /* ... */ }

        private void remove(ThreadLocal key) { /* ... */ }

        // ...
    }
}
```

可见 key 的实质是 ThreadLocal 的弱引用，value 的实质就是 Entry 实例中保存的对象。

每个 Thread 实例中包含一个 ThreadLocalMap 对象`threadLocals`。在不同线程访问同一个 ThreadLocal 对象时，其`set()`方法会以此对象为 key，以实际保存的值为 value，保存到各自线程的`threadLocals`对象中。

总之，ThreadLocal 最终的操作对象，就是当前线程的`threadLocals`对象中的 Entry 数组`table`，因此在不同线程中访问同一个 ThreadLocal 对象，它们的读写操作仅限于各线程内部。

### MessageQueue 消息队列的工作原理

Message 中含有一个 Message 类型的变量`next`，因此 Message 本身可看作链表结点，MessageQueue 正是以 Message 链表为存储结构，将其封装为队列的逻辑结构。

MessageQueue 中有一个 Message 类型变量`mMessages`，就是链表的头结点。在`mMessages`链表中，元素的`when`属性越小，越排在链表前面。

要理解 MessageQueue，要先了解两种特殊的 Message：Asynchronous Message（异步消息）和 Sync Barrier（同步障碍）。

#### Asynchronous Message 异步消息

网上一些文章直接把 Message 叫做异步消息，这种说法是有歧义的。

Message 默认是同步（synchronous）的，要修改 Message 同步或异步（asynchronous）标识，可调用 Message 的`setAsynchronous()`方法（此接口在 API 22 中开放）。

默认情况下，Handler 也都是同步的，除非使用 Handler 类带有 boolean 类型`async`形参的构造方法来构造实例，将自己的`mAsynchronous`属性置为`true`，而目前（Android N）这些构造方法都用`@hide`修饰，对第三方 APP 隐藏了接口。

当一个 Handler 对象发送消息时，不管是 post 方法还是 send 方法，最终都是调用`enqueueMessage`方法将 Message 加入队列：

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

可见只要 Handler 本身是异步的，其 Message 加入队列时，都会先将 Message 也置为异步。

#### Sync Barrier 同步障碍

Barrier，即 Sync Barrier，字面意义为障碍或同步障碍，本质上是一种特殊的 Message。MessageQueue 提供了加入 Barrier 的隐藏方法：

```
/**
 * @hide
 */
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    // Enqueue a new sync barrier token.
    // We don't need to wake the queue because the purpose of a barrier is to stall it.
    synchronized (this) {
        final int token = mNextBarrierToken++;
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { // invariant: p == prev.next
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

发送一个 Barrier 的过程，就是获取一个 Message 新实例，然后将其插入到 MessageQueue 中，使队列按照元素`when`的值升序排列。由于没有设置 Barrier 的`target`属性，故`target`属性为`null`。可以认为，`target`为`null`的 Message 就是 Barrier。

发送 Barrier 得到的返回值是一个 int 类型的 token，依据此 token，可以使用`removeSyncBarrier()`方法从队列中移除对应的 Barrier。

Barrier 可用于立即推迟接下来的同步消息的处理，直到此 Barrier 被移除。但异步消息不受 Barrier 的影响，继续正常处理。具体如何实现，可参考下面`next()`方法部分。

#### MessageQueue 的出队和入队

`enqueueMessage()`方法是 MessageQueue 的入队接口，`next()`方法是出队接口。`next()`方法会遍历 Message 链表，并取出到时的 Message，方法中包含一个无限循环，不断轮询链表中的 Message，如果没有到时的 Message，则`next()`方法会阻塞，`mBlocked`就用来指示`next()`方法是否已阻塞。

`enqueueMessage()`方法入队逻辑的主要代码如下，其中`msg`和`when`是传入的参数，分别为 Message 对象和其处理时间：

```
msg.markInUse();
msg.when = when;
Message p = mMessages;
boolean needWake;
if (p == null || when == 0 || when < p.when) {
    // New head, wake up the event queue if blocked.
    msg.next = p;
    mMessages = msg;
    needWake = mBlocked;
} else {
    // Inserted within the middle of the queue.  Usually we don't have to wake
    // up the event queue unless there is a barrier at the head of the queue
    // and the message is the earliest asynchronous message in the queue.
    needWake = mBlocked && p.target == null && msg.isAsynchronous();
    Message prev;
    for (;;) {
        prev = p;
        p = p.next;
        if (p == null || when < p.when) {
            break;
        }
        if (needWake && p.isAsynchronous()) {
            needWake = false;
        }
    }
    msg.next = p; // invariant: p == prev.next
    prev.next = msg;
}
```

入队逻辑很清晰：

- 如果入队 Message 是队列中最早（`when`属性最小）的，直接用头插法插入链表即可
    - 若`next()`方法被阻塞，需要立即处理入队 Message，显然需要唤醒`next()`方法
- 否则，需要将入队 Message 插入队列中合适的位置，使队列按照元素的`when`属性升序排列，此时遍历链表，找到第一个`when`大于当前入队消息的结点，并插入到此结点之前
    - 若原头结点是 Barrier，说明`next()`方法已被阻塞或即将阻塞
        - 如果入队 Message 是队列中最早的异步消息，则需要唤醒`next()`方法
        - 否则，不需要唤醒（唤醒的任务由`next()`方法进行）

`next()`方法主要出队逻辑如下：

```
// Try to retrieve the next message.  Return if found.
final long now = SystemClock.uptimeMillis();
Message prevMsg = null;
Message msg = mMessages;
if (msg != null && msg.target == null) {
    // Stalled by a barrier.  Find the next asynchronous message in the queue.
    do {
        prevMsg = msg;
        msg = msg.next;
    } while (msg != null && !msg.isAsynchronous());
}
if (msg != null) {
    if (now < msg.when) {
        // Next message is not ready.  Set a timeout to wake up when it is ready.
        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
    } else {
        // Got a message.
        mBlocked = false;
        if (prevMsg != null) {
            prevMsg.next = msg.next;
        } else {
            mMessages = msg.next;
        }
        msg.next = null;
        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
        msg.markInUse();
        return msg;
    }
} else {
    // No more messages.
    nextPollTimeoutMillis = -1;
}
```

第一个 if 语句判断头结点是否是一个 Barrier，如果是，说明队列中同步消息的处理被推迟，则需要找到第一个异步消息。

此时，`msg`或者指向队列中下一个需要处理的消息（可能是同步的，也可能是异步的），或者为`null`。

`nextPollTimeoutMillis`表示下次轮询的时间。

第二个 if 语句中，

- 如果`msg != null`判定为真，表示有消息待处理
    - 若消息的`when`还没到，则计算还需要等待的时间，下次循环时，先调用 native 方法`nativePollOnce()`阻塞相应的时间
    - 否则说明消息需要立即处理，消息出队
- 否则表示队列中没有需要处理的消息

#### MessageQueue 的退出

消息队列使用`quit(boolean safe)`方法退出，`safe`参数用于指示是否安全退出：

```
if (safe) {
    removeAllFutureMessagesLocked();
} else {
    removeAllMessagesLocked();
}
```

`removeAllMessagesLocked()`方法会直接移除回收队列中所有的 Message：

```
private void removeAllMessagesLocked() {
    Message p = mMessages;
    while (p != null) {
        Message n = p.next;
        p.recycleUnchecked();
        p = n;
    }
    mMessages = null;
}
```

`removeAllFutureMessagesLocked()`方法会移除未到时的 Message，已经到时的 Message 依然会处理：

```
private void removeAllFutureMessagesLocked() {
    final long now = SystemClock.uptimeMillis();
    Message p = mMessages;
    if (p != null) {
        if (p.when > now) {
            removeAllMessagesLocked();
        } else {
            Message n;
            for (;;) {
                n = p.next;
                if (n == null) {
                    return;
                }
                if (n.when > now) {
                    break;
                }
                p = n;
            }
            p.next = null;
            do {
                p = n;
                n = p.next;
                p.recycleUnchecked();
            } while (n != null);
        }
    }
}
```

### Looper 的工作原理

Looper 在一个线程中的调用过程很简单，先使用`Looper.prepare()`为当前线程创建一个 Looper，然后通过`Looper.loop()`来开启消息循环。例如：

```
new Thread() {
    @Override
    public void run() {
        Looper.prepare();
        Handler handler = new Handler();
        Looper.loop();
    };
}.start();
```

如果没有创建 Looper，则创建 Handler 时会抛出异常。
