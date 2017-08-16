# 在 Android Java libcore 核心库打印 Log

Android Java 核心库中是无法直接使用 android.util.Log 的，添加后编译不通过，因为 framework 中的 Java API 依赖于 Java 核心库。

本文以`libcore/ojluni/src/main/java/java/io/File.java`为例，简单介绍在核心库中打印 Log 的几种方法。

## 使用 System.out 和 System.err

这是很常见的方法，在 Android 中它被重定向到本地的 Log 系统，tag 分别为`System.out`和`System.err`。

缺点在于，它不能自定义 tag，而且需要注意的是，这种方法是在 SystemServer 进程创建之后、启动之前进行重定向的，在这之前无法打印 Log。

```java
// frameworks/base/core/java/com/android/internal/os/RuntimeInit.java

/**
 * The main function called when started through the zygote process. This
 * could be unified with main(), if the native code in nativeFinishInit()
 * were rationalized with Zygote startup.<p>
 *
 * Current recognized args:
 * <ul>
 *   <li> <code> [--] &lt;start class name&gt;  &lt;args&gt;
 * </ul>
 *
 * @param targetSdkVersion target SDK version
 * @param argv arg strings
 */
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
        throws ZygoteInit.MethodAndArgsCaller {
    if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");

    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "RuntimeInit");
    redirectLogStreams();

    commonInit();
    nativeZygoteInit();
    applicationInit(targetSdkVersion, argv, classLoader);
}

// ...

/**
 * Redirect System.out and System.err to the Android log.
 */
public static void redirectLogStreams() {
    System.out.close();
    System.setOut(new AndroidPrintStream(Log.INFO, "System.out"));
    System.err.close();
    System.setErr(new AndroidPrintStream(Log.WARN, "System.err"));
}
```

在`redirectLogStreams()`方法中，设置输出流为 AndroidPrintStream，当使用`System.out`和`System.err`进行打印时，最终是调用的是 android.util.Log：

```java
// frameworks/base/core/java/com/android/internal/os/AndroidPrintStream.java

/**
 * Print stream which log lines using Android's logging system.
 *
 * {@hide}
 */
class AndroidPrintStream extends LoggingPrintStream {

    private final int priority;
    private final String tag;

    /**
     * Constructs a new logging print stream.
     *
     * @param priority from {@link android.util.Log}
     * @param tag to log
     */
    public AndroidPrintStream(int priority, String tag) {
        if (tag == null) {
            throw new NullPointerException("tag");
        }

        this.priority = priority;
        this.tag = tag;
    }

    protected void log(String line) {
        Log.println(priority, tag, line);
    }
}
```

## 使用 java.util.logging.Logger

Java 核心库中有 java.util.logging.Logger，在 Android 中它也被重定向到 Android 本地的 Log 系统。

使用方法很简单，在需要打印 Log 的源码中添加：

```java
// libcore/ojluni/src/main/java/java/io/File.java

private static final Logger sLogger = Logger.getLogger("File");

private static void logi(String msg) {
    sLogger.info(msg);
}
```

打印 Log 时只需调用`logi()`方法即可。

这里使用的是 Level 为 INFO 的 Log。你也可以自定义 Level，核心库 java.util.logging.Logger 与 Android 本地 android.util.Log 的 Level 对应关系，可以参考 java.util.logging.Level 和 com.android.internal.logging.AndroidHandler。

和`System.out`一样，这种方法也是在 SystemServer 进程创建之后、启动之前进行重定向的，在这之前无法打印 Log。

事实上，这里 Log 的打印实际上是调用了 Logger 中注册的 Handler，这里的 Handler 是 java.util.logging.Handler，不是 android.os.Handler。

Handler 的注册，紧随`RuntimeInit.zygoteInit()`方法中`redirectLogStreams()`后，调用`commonInit()`方法：

```java
// frameworks/base/core/java/com/android/internal/os/RuntimeInit.java

private static final void commonInit() {

    // ...

    /*
     * Sets handler for java.util.logging to use Android log facilities.
     * The odd "new instance-and-then-throw-away" is a mirror of how
     * the "java.util.logging.config.class" system property works. We
     * can't use the system property here since the logger has almost
     * certainly already been initialized.
     */
    LogManager.getLogManager().reset();
    new AndroidConfig();

    // ...

}
```

这里实例化了 AndroidConfig 然后丢弃。

```java
// frameworks/base/core/java/com/android/internal/logging/AndroidConfig.java

/**
 * Implements the java.util.logging configuration for Android. Activates a log
 * handler that writes to the Android log.
 */
public class AndroidConfig {

    /**
     * This looks a bit weird, but it's the way the logging config works: A
     * named class is instantiated, the constructor is assumed to tweak the
     * configuration, the instance itself is of no interest.
     */
    public AndroidConfig() {
        super();

        try {
            Logger rootLogger = Logger.getLogger("");
            rootLogger.addHandler(new AndroidHandler());
            rootLogger.setLevel(Level.INFO);

            // Turn down logging in Apache libraries.
            Logger.getLogger("org.apache").setLevel(Level.WARNING);
        } catch (Exception ex) {
            ex.printStackTrace();
        }
    }
}
```

实例化过程中注册了 AndroidHandler。

当使用 Logger 打印 Log 时，最终调用的是`Logger#log(LogRecord)`方法：

```java
// libcore/ojluni/src/main/java/java/util/logging/Logger.java

public void log(LogRecord record) {

    // ...

    Logger logger = this;
    while (logger != null) {
        for (Handler handler : logger.getHandlers()) {
            handler.publish(record);
        }

        if (!logger.getUseParentHandlers()) {
            break;
        }

        logger = logger.getParent();
    }
}
```

对已经注册的 Handler 回调其`publish()`方法：

```java
// frameworks/base/core/java/com/android/internal/logging/AndroidHandler.java

@Override
public void publish(Logger source, String tag, Level level, String message) {
    // TODO: avoid ducking into native 2x; we aren't saving any formatter calls
    int priority = getAndroidLevel(level);
    if (!Log.isLoggable(tag, priority)) {
        return;
    }

    try {
        Log.println(priority, tag, message);
    } catch (RuntimeException e) {
        Log.e("AndroidHandler", "Error logging message.", e);
    }
}
```

可见最终还是调用的 android.util.Log。

如果需要在 SystemServer 之前打印 Log，则此方法无效。

## 移植 android.util.Log

## 打印栈信息 Stack Trace
