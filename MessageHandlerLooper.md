# Android 的消息机制
## 概述
Android 的 UI 控件不是线程安全的，因此规定访问 UI 必须在主线程（即 UI 线程）中进行。ViewRootImpl 的 checkThread 方法对 UI 操作进行验证，如果在后台线程中访问 UI，程序就会抛出异常。

但主线程中不适合进行耗时操作，因为这样会导致 ANR，所以耗时操作应在后台线程中进行。执行完成后，再使用 Handler 切换到主线程进行 UI 操作。

为什么 UI 控件不设计成线程安全的呢？一是避免访问逻辑变复杂，二是避免效率低下。

Handler 是 Android 消息机制的上层接口，通过它可以将一个任务切换到 Handler 所在的线程中执行。Handler 的运行需要底层的 MessageQueue 和 Looper 支撑。MessageQueue 维护了一个 Message 链表，以队列的逻辑结构向 Handler 和 Looper 提供接口。Looper 中维护了一个 MessageQueue 类型的消息队列，Handler 将 Message 加入队列，Looper 从队列中取出 Message，Looper 以无限循环的方式查找是否有新 Message。Looper 类中 ThreadLocal<Looper> 类型的静态变量`sThreadLocal`，用于为当前线程保存和获取 Looper（ThreadLocal 用于为当前线程保存变量，此变量只能被当前线程访问，其它线程无法访问）。

Handler 创建时，如果不指定 Looper，默认会采用当前线程的 Looper 来构造消息循环系统：
```
mLooper = Looper.myLooper();
```
线程默认是没有 Looper 的，如果不为线程创建 Looper，使用 Handler 时就会抛出异常。

主线程，即 ActivityThread，其创建时就会初始化 Looper，因此主线程中默认可以使用 Handler。

Handler 创建完毕后，可使用 post 方法将一个 Runnable，也可使用 send 方法发送一个 Message，本质都是将一个 Message 加入 Handler 所在线程对应的 Looper 中的消息队列。Looper 发现有新 Message，就会调用 dispatchMessage 来处理。因此 Handler 中的业务逻辑就在创建 Handler 所在的线程中执行。这也是后台线程能通过主线程中的 Handler 来更新 UI 的原因。

## Android 消息机制分析
### ThreadLocal 工作原理
ThreadLocal 是一个线程内部的数据存储类，只有在指定线程中可以获取到存储的数据，其它线程无法获取。不同线程访问同一个 ThreadLocal 对象，它们对 ThreadLocal 的读写仅限于各自线程的内部，因此不同线程中维护一套数据副本且彼此互不干扰。Looper、ActivityThread、ActivityManagerService 中都用到了 ThreadLocal。

ThreadLocal 的原理，在 Android 7.0 之前使用的是一个 Values 内部类，7.0 中改为 ThreadLocalMap 类，其本质是一个定制化的哈希表。ThreadLocalMap 对外是以 ThreadLocal 对象为 key，以需要保存的对象为 value，提供存取键值对的接口，实际上它维护了一个 Entry 类型的数组，这些键值对就保存在 Entry 对象中：
```
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```
可见 key 的实质是 ThreadLocal 的弱引用，value 的实质就是保存的对象。
