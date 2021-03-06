# Android 的消息机制

## 概述

Android 的 UI 控件不是线程安全的，因此规定访问 UI 必须在主线程（即 UI 线程）中进行。ViewRootImpl 的`checkThread()`方法对 UI 操作进行验证，如果在后台线程中访问 UI，程序就会抛出异常。

但主线程中不适合进行耗时操作，因为这样会导致 ANR，所以耗时操作应在后台线程中进行。执行完成后，再使用 Handler 切换到主线程进行 UI 操作。

为什么 UI 控件不设计成线程安全的呢？一是避免访问逻辑变复杂，二是避免效率低下。

Handler 是 Android 消息机制的上层接口，通过它可以将一个任务切换到 Handler 所在的线程中执行。Handler 的运行需要 MessageQueue 和 Looper 支撑。MessageQueue 维护了一个 Message 链表，以队列的逻辑结构向 Handler 和 Looper 提供接口，Looper 中维护了一个 MessageQueue 类型的消息队列。Handler 将 Message 加入队列，Looper 以无限循环的方式查找队列中是否有需要处理的 Message。

Looper 类中 ThreadLocal<Looper> 类型的静态变量`sThreadLocal`，用于为当前线程保存和获取 Looper（ThreadLocal 用于为当前线程保存变量，此变量只能被当前线程访问，其它线程无法访问）。

后台线程默认是没有初始化 Looper 的。创建 Handler 实例时，如果没有事先为当前线程创建 Looper，或没有使用特定的构造方法指定 Looper 对象，就会抛出异常。后台线程中一般使用`Looper.prepare()`方法为当前线程创建一个 Looper。

主线程，即 ActivityThread，其创建时就会初始化 Looper，因此主线程中默认可以使用 Handler。

Handler 创建完毕后，可使用 post 开头的方法（以下简称 post 方法）发送一个 Runnable，也可使用 send 开头的方法（以下简称 send 方法）发送一个 Message，本质都是将一个 Message 加入 Handler 对应的 Looper 中的消息队列。post 方法本质上是将发送的 Runnable 指定为新建 Message 实例的`callback`属性，然后再将此 Message 加入队列。Looper 发现有需要处理的 Message，就会调用对应 Handler 的`dispatchMessage()`方法来处理。因此 Handler 中的业务逻辑就在 Handler 对应的 Looper 所在线程中执行。这也是后台线程能通过主线程中的 Handler 来更新 UI 的原因。

## Android 消息机制分析

### ThreadLocal 工作原理

ThreadLocal 类用于维护线程作用域的变量，不同线程访问同一个 ThreadLocal 对象时，它们对 ThreadLocal 的读写仅限于各自线程的内部，即不同线程中维护一套数据副本，彼此互不干扰。Looper、ActivityThread、ActivityManagerService 中都用到了 ThreadLocal。

在开始采用 OpenJDK 的 Android 7.0 中，ThreadLocal 的原理是使用了一个静态嵌套类 ThreadLocalMap（一个定制化的哈希表）。

每个 Thread 实例中都包含一个 ThreadLocalMap 对象`threadLocals`：

```
/* ThreadLocal values pertaining to this thread. This map is maintained
 * by the ThreadLocal class. */
ThreadLocal.ThreadLocalMap threadLocals = null;
```

ThreadLocalMap 以 ThreadLocal 对象为 key，以需要保存的对象为 value，提供存取键值对的接口。实际上 ThreadLocalMap 维护了一个 ThreadLocal.ThreadLocalMap.Entry 类型的数组`table`，这些键值对就保存在 Entry 数组中。ThreadLocal 类结构如下：

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

在不同线程访问同一个 ThreadLocal 对象时，其`set()`方法会以此对象为 key，以实际保存的值为 value，保存到各自线程的`threadLocals`对象中。因此在不同线程中访问同一个 ThreadLocal 对象，它们的读写操作仅限于各线程内部。

### MessageQueue 消息队列的工作原理

Message 中含有一个 Message 类型的变量`next`，因此 Message 本身可看作链表结点，MessageQueue 正是以 Message 链表为存储结构，将其封装为队列的逻辑结构。

MessageQueue 中有一个 Message 类型变量`mMessages`，就是链表的头结点。在`mMessages`链表中，元素的`when`属性越小，越排在链表前面。

要理解 MessageQueue，要先了解两种特殊的 Message：Asynchronous Message（异步消息）和 Sync Barrier（同步障碍）。

#### Asynchronous Message 异步消息

网上一些文章直接把 Message 叫做异步消息，这种说法是有歧义的。

Message 默认是同步（synchronous）的，要修改 Message 同步或异步（asynchronous）标识，可调用 Message 的`setAsynchronous()`方法（此接口在 API 22 中开放）。

默认情况下，Handler 也都是同步的，除非使用 Handler 类带有 boolean 类型形参`async`的构造方法来构造实例，将自己的`mAsynchronous`属性置为`true`，而目前（Android N）这些构造方法都用`@hide`修饰，对第三方 APP 隐藏了接口。

当一个 Handler 对象发送消息时，不管是 post 方法还是 send 方法，最终都是调用`enqueueMessage`方法将 Message 加入队列：

```
// frameworks/base/core/java/android/os/MessageQueue.java

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
// frameworks/base/core/java/android/os/MessageQueue.java

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
// frameworks/base/core/java/android/os/MessageQueue.java

boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

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

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

入队逻辑很清晰，直接看中间的`if else`块：

- 如果入队 Message 是队列中最早（`when`属性最小）的，直接用头插法插入链表即可
    - 若`next()`方法被阻塞，需要立即处理入队 Message，显然需要唤醒`next()`方法
- 否则，需要将入队 Message 插入队列中合适的位置，使队列按照元素的`when`属性升序排列，此时遍历链表，找到第一个`when`大于当前入队消息的结点，并插入到此结点之前
    - 若原头结点是 Barrier，说明`next()`方法已被阻塞或即将阻塞
        - 如果入队 Message 是队列中最早的异步消息，则需要唤醒`next()`方法
        - 否则，不需要唤醒（唤醒的任务由`next()`方法进行）

`next()`方法主要出队逻辑如下：

```
// frameworks/base/core/java/android/os/MessageQueue.java

Message next() {
    // Return here if the message loop has already quit and been disposed.
    // This can happen if the application tries to restart a looper after quit
    // which is not supported.
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    // ...

    int nextPollTimeoutMillis = 0;
    for (;;) {

        // ...

        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
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

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // ...

        }

        // ...

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

`nextPollTimeoutMillis`指示下次轮询的时间，使用本地方法`nativePollOnce(long ptr, int timeoutMillis)`来阻塞相应的时间。如果`nextPollTimeoutMillis`为 -1，说明没有需要处理的消息，进入无限阻塞，直到有新的消息为止。

第一个 if 语句判断头结点是否是一个 Barrier，如果是，说明队列中同步消息的处理被推迟，则需要找到第一个异步消息。

此时，`msg`或者指向队列中下一个需要处理的消息（可能是同步的，也可能是异步的），或者为`null`。

第二个 if 语句中，

- 如果`msg != null`，表示有消息待处理
    - 若消息的`when`还没到，则计算还需要等待的时间，下次轮询时，先调用 native 方法`nativePollOnce()`阻塞相应的时间
    - 否则说明消息需要立即处理，消息出队
- 否则表示队列中没有需要处理的消息，将 nextPollTimeoutMillis 设置为 -1，进入无限阻塞，直到有新的消息为止

上面的`next()`源码省略了一些 MessageQueue.IdleHandler 相关的代码，此接口用于在线程没有当前需要处理的 Message、即将阻塞时，处理一些事情，在此不赘述。如果没有需要处理的 IdleHandler，则会使用`continue;`语句进行下个轮询，进入阻塞；反之则进行回调，for 循环的最后，nextPollTimeoutMillis 被重新设为 0，这是因为在回调过程中，可能有新的 Message 入队。

总得来看，如果已经退出消息循环，则`next()`方法返回`null`，否则返回下个需处理的 Message 对象。

#### MessageQueue 的退出

构造 MessageQueue 时，`mQuitAllowed`属性标识消息队列是否允许退出：

```
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();
}
```

后台线程允许退出，而主线程是不允许的，否则会抛出异常。消息队列的退出使用`quit(boolean safe)`方法，`safe`参数用于指示是否安全退出：

```
// frameworks/base/core/java/android/os/MessageQueue.java

void quit(boolean safe) {
    if (!mQuitAllowed) {
        throw new IllegalStateException("Main thread not allowed to quit.");
    }

    synchronized (this) {
        if (mQuitting) {
            return;
        }
        mQuitting = true;

        if (safe) {
            removeAllFutureMessagesLocked();
        } else {
            removeAllMessagesLocked();
        }

        // We can assume mPtr != 0 because mQuitting was previously false.
        nativeWake(mPtr);
    }
}
```

`removeAllMessagesLocked()`方法会直接移除回收队列中所有的 Message：

```
// frameworks/base/core/java/android/os/MessageQueue.java

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
// frameworks/base/core/java/android/os/MessageQueue.java

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

在后台线程中，Looper 和 Handler 的调用过程很简单。一种常用的方法是先使用`Looper.prepare()`为当前线程创建一个 Looper，并实例化一个 Handler，然后通过`Looper.loop()`来开启消息循环。例如：

```
class LooperThread extends Thread {
    public Handler mHandler;

    public void run() {
        Looper.prepare();

        mHandler = new Handler() {
            public void handleMessage(Message msg) {
                // process incoming messages here
            }
        };

        Looper.loop();
    }
}
```

另一种方法是在构造 Handler 实例时指定 Looper，即在构造方法中传入一个 Looper 对象。

总之，创建 Handler 时，必须有一个 Looper 对象与其对应，否则会抛出异常，详见下面 Handler 部分。

#### Looper 的创建

在后台线程中，使用

```
Looper.prepare();
```

为当前线程创建 Looper。`prepare()`方法实现如下：

```
// frameworks/base/core/java/android/os/Looper.java

// sThreadLocal.get() will return null unless you've called prepare().
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();

// ...

public static void prepare() {
    prepare(true);
}

private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}

// ...

private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

根据上面 ThreadLocal 原理的分析，`sThreadLocal`会为当前线程生成一个副本。线程在调用`Looper.prepare()`时，构造一个 Looper 实例，并初始化消息队列`mQueue`。此 Looper 保存于`sThreadLocal`中，作用域为当前线程。`quitAllowed`参数指示消息队列是否允许手动退出（参考上面“MessageQueue 的退出”），可见后台线程是允许退出消息队列的。

而主线程使用：

```
// frameworks/base/core/java/android/os/Looper.java

private static Looper sMainLooper;  // guarded by Looper.class

// ...

public static void prepareMainLooper() {
    prepare(false);
    synchronized (Looper.class) {
        if (sMainLooper != null) {
            throw new IllegalStateException("The main Looper has already been prepared.");
        }
        sMainLooper = myLooper();
    }
}

public static Looper getMainLooper() {
    synchronized (Looper.class) {
        return sMainLooper;
    }
}

// ...

public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

来初始化 Looper（不应手动调用），`prepare(quitAllowed)`方法参数为`false`，因此主线程不允许手动退出消息队列。

注意到主线程的 Looper 保存在一个静态的`sMainLooper`对象中，而没有使用 ThreadLocal，这是因为一个进程对应一个 Dalvik/ART 虚拟机，一个进程也只包含一个主线程，故同一进程中获取到的是同一个`sMainLooper`对象，不存在多线程产生对象副本的问题。

#### Looper.loop() 开启消息循环

```
// frameworks/base/core/java/android/os/Looper.java

public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    // ...

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // ...

        try {
            msg.target.dispatchMessage(msg);
        } finally {

            // ...

        }

        // ...

        msg.recycleUnchecked();
    }
}
```

`loop()`方法主要的工作就是不断循环地取出下一个需处理的 Message 并进行处理。其大致流程：

1. 调用 MessageQueue 的`next()`方法，从消息队列中获取下一个需要处理的 Message，此过程可能会阻塞

    - 如果获取到的 Message 为`null`，从上面 MessageQueue 的`next()`方法可知，消息队列已经退出，故直接返回

2. 获取到 Message 后，回调其对应的 Handler（即 Message 的`target`属性）的`dispatchMessage()`方法，因此 Handler 中的业务逻辑是在其 Looper 对象所在线程中执行的，这就实现了线程切换

#### 退出消息循环

Looper 提供了`quit()`方法和`quitSafely()`方法来退出，调用的分别是 MessageQueue 的`quit(false)`方法和`quit(true)`方法。

### Handler 的工作原理

#### Handler 的创建

创建 Handler 时，如果不显式指定 Looper 对象，则最终会调用如下构造方法：

```
// frameworks/base/core/java/android/os/Handler.java

/**
 * @hide
 */
public Handler(Callback callback, boolean async) {

    // ...

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }

    // ...

}
```

Handler 会试图获取当前线程的 Looper 对象，这也表示 Handler 的回调逻辑会在构造它的线程中执行。

如果显式指定 Looper 对象，则最终会调用如下构造方法：

```
/**
 * @hide
 */
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

总之，Handler 的回调逻辑最终会在此 Looper 对象所在线程中执行。

#### 消息发送

Handler 通过 post 方法和 send 方法来发送消息。post 方法会构造一个 Message 对象，并将发送的 Runnable 封装为此 Message 的`callback`属性，最终也是通过 send 方法来实现消息发送的。发送的消息会被插入 Handler 所关联的 Looper 对象中的消息队列`mQueue`中，如：

```
// frameworks/base/core/java/android/os/Handler.java

public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

可见最终调用的是 MessageQueue 的`enqueueMessage()`方法。

#### 消息处理

在`Looper.loop()`开启消息循环后，每当获取到需要处理的 Message，则调用`msg.target.dispatchMessage(msg)`来执行对应 Handler 中的业务逻辑。`dispatchMessage()`方法处理逻辑如下：

```
// frameworks/base/core/java/android/os/Handler.java

/**
 * Callback interface you can use when instantiating a Handler to avoid
 * having to implement your own subclass of Handler.
 *
 * @param msg A {@link android.os.Message Message} object
 * @return True if no further handling is desired
 */
public interface Callback {
    public boolean handleMessage(Message msg);
}

/**
 * Subclasses must implement this to receive messages.
 */
public void handleMessage(Message msg) {
}

/**
 * Handle system messages here.
 */
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}

// ...

private static void handleCallback(Message message) {
    message.callback.run();
}
```

- 如果`callback`属性不为`null`，则直接执行它
- 否则
    - 如果此 Handler 的`mCallback`不为`null`，则回调`mCallback`的`handleMessage()`方法
    - 否则回调 Handler 的`handleMessage()`方法

Message 的`callback`属性就是使用 post 方法发送的 Runnable 对象。

`mCallback`是 Handler.Callback 接口类型对象。

创建 Handler 最常见的方式是派生一个 Handler 的子类并重写`handleMessage()`方法，如果不想派生子类，就可以通过 Callback 接口来实现。

### 主线程的消息循环

ActivityThread 并没有继承 Thread，严格来说它并不是线程，但它运行在主线程中，其`main()`方法是主线程的入口，因此常把 ActivityThread 称作主线程。

事实上，主线程是在 ActivityManagerService 的 `startProcessLocked()`方法中，调用`Process.start()`方法启动的，它会在当前新建的线程中加载 ActivityThread 类，并调用`ActivityThread.main()`方法作为应用程序的入口点。

`ActivityThread.main()`方法中会调用`Looper.prepareMainLooper()`创建主线程的 Looper，实例化 ActivityThread，并开启消息循环：

```
// frameworks/base/core/java/android/app/ActivityThread.java

// ...

final Looper mLooper = Looper.myLooper();

// ...

public static void main(String[] args) {

    // ...

    Looper.prepareMainLooper();

    ActivityThread thread = new ActivityThread();

    // ...

    Looper.loop();

    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

处理消息还需要一个 Handler，即内部类 H：

```
// frameworks/base/core/java/android/app/ActivityThread.java

final H mH = new H();

// ...

private class H extends Handler {
    public static final int LAUNCH_ACTIVITY         = 100;
    public static final int PAUSE_ACTIVITY          = 101;
    public static final int PAUSE_ACTIVITY_FINISHING= 102;
    public static final int STOP_ACTIVITY_SHOW      = 103;
    public static final int STOP_ACTIVITY_HIDE      = 104;

    // ...

    public void handleMessage(Message msg) {
        if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
        switch (msg.what) {
            case LAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                r.packageInfo = getPackageInfoNoCheck(
                        r.activityInfo.applicationInfo, r.compatInfo);
                handleLaunchActivity(r, null, "LAUNCH_ACTIVITY");
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;
            case RELAUNCH_ACTIVITY: {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");
                ActivityClientRecord r = (ActivityClientRecord)msg.obj;
                handleRelaunchActivity(r);
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            } break;

            // ...

        }

        // ...

    }

    // ...

}
```

H 中定义了一组消息类型，包含四大组件的启动、停止等过程。
