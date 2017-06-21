# Activity 的启动原理及 ActivityManagerService 的启动

通常启动一个 Activity 十分简单，例如从当前的 Activity 启动 TargetActivity，一个典型的显式 Intent 启动方式如下：

```java
Intent intent = new Intent(this, TargetActivity.class);
startActivity(intent);
```

但 Activity 启动过程的具体实现比较复杂。本文以 Android 7.1.1 源码为例，分析 Activity 启动的大体流程。

## Activity#startActivity()

从`frameworks/base/core/java/android/app/Activity.java`源码看，启动 Activity 有很多种方法，实现的原理大同小异，下面以`startActivity(Intent)`为例说明 Activity 的启动过程。

`startActivity(Intent)`最终调用的是`startActivityForResult(Intent, int, Bundle)`方法：

```java
// frameworks/base/core/java/android/app/Activity.java

public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {

        // ...

        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);

        // ...

    } else {

        // ...

    }
}
```

`mParent`也是 Activity 类型，它原先用于在一个界面中嵌入多个 Activity，后来被 Fragment 代替，故这里只关注`mParent == null`这部分逻辑。

`mInstrumentation`是 Instrumentation 类型的对象，这个类主要用于监听系统和应用的交互。

`mMainThread`是 ActivityThread 类型对象。ActivityThread 运行在主线程中，其`main()`方法是主线程的入口，因此常把 ActivityThread 称作主线程，`mMainThread`就代表了当前应用的主线程。例如在 Launcher 中点击图标启动一个应用的 Activity，`mMainThread`就是 Launcher 的主线程。`mMainThread.getApplicationThread()`获取的是一个 ApplicationThread 类型的对象，ApplicationThread 是 ActivityThread 的一个内部类。而 ApplicationThread 继承了 ApplicationThreadNative，这是一个 Binder。显然，这里获取到的 ApplicationThread 对象是一个运行在 Launcher 主线程中的 Binder 服务，利用它来进行进程间通信。

`mToken`是一个 Binder 对象的远程接口。

## Instrumentation#execStartActivity()

```java
// frameworks/base/core/java/android/app/Instrumentation.java

public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {

    // ...

    try {

        // ...

        int result = ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

这里`ActivityManagerNative.getDefault()`获取的是一个 IActivityManager 客户端代理，利用它和系统中的 ActivityManagerService 进行通信。

## ActivityManagerNative.getDefault()

抽象类 ActivityManagerNative 是一个 Binder，它实现了 IActivityManager 接口。静态方法`ActivityManagerNative.getDefault()`返回的是 IActivityManager 实例：

```java
// frameworks/base/core/java/android/app/ActivityManagerNative.java

/** {@hide} */
public abstract class ActivityManagerNative extends Binder implements IActivityManager
{

    // ...

    /**
     * Retrieve the system's default/global activity manager.
     */
    static public IActivityManager getDefault() {
        return gDefault.get();
    }

    // ...

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
}
```

它使用延迟初始化的方式对外提供单例：

```java
// frameworks/base/core/java/android/util/Singleton.java

/**
 * Singleton helper class for lazily initialization.
 *
 * Modeled after frameworks/base/include/utils/Singleton.h
 *
 * @hide
 */
public abstract class Singleton<T> {
    private T mInstance;

    protected abstract T create();

    public final T get() {
        synchronized (this) {
            if (mInstance == null) {
                mInstance = create();
            }
            return mInstance;
        }
    }
}
```

只要继承 Singleton<T> 并实现`create()`方法即可实现单例模式，在需要经常实现单例模式时，不失为一个便捷的方法。

通过`ServiceManager.getService("activity")`即可获取其单例，这里获取到的是一个 IActivityManager 的客户端 Binder，也就是 ActivityManagerProxy 对象。

Android 系统中，ActivityManagerService 继承了 ActivityManagerNative，是 IActivityManager 的具体实现。那么并作为服务端 Binder，它是如何启动的？

## ActivityManagerService 的启动

Zygote 进程是所有 Android 应用程序进程的父进程，Zygote 在启动过程中，会根据启动参数 fork 生成第一 Java 进程，它就是 SystemServer 进程，也就是使用`ps`命令看到的名为`system_server`的进程，它运行 Android 系统的绝大部分服务，包括 ActivityManagerService。这里暂不讨论 Zygote 是如何 fork SystemServer 进程的。

SystemServer 进程的入口是它的静态方法`main()`：

```java
// frameworks/base/services/java/com/android/server/SystemServer.java

/**
 * The main entry point from zygote.
 */
public static void main(String[] args) {
    new SystemServer().run();
}

// ...

private void run() {
    try {

        // ...

        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
    }

    // ...

    // Start services.
    try {
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
        startBootstrapServices();
        startCoreServices();
        startOtherServices();
    }

    // ...
}

// ...

/**
 * Starts the small tangle of critical services that are needed to get
 * the system off the ground.  These services have complex mutual dependencies
 * which is why we initialize them all in one place here.  Unless your service
 * is also entwined in these dependencies, it should be initialized in one of
 * the other functions.
 */
private void startBootstrapServices() {

    // ...

    // Activity manager runs the show.
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();

    // ...

    // Set up the Application instance for the system process and get started.
    mActivityManagerService.setSystemProcess();

    // ...

}
```

创建了 SystemServiceManager 实例`mSystemServiceManager`，并注册到 LocalServices 中。

LocalServices 本质就是在一个 ArrayMap 中进行存取，在此不再赘述。它的使用与 ServiceManager 相似，但注册的服务并不是 Binder 对象，因此只在同一进程中使用。

然后在`startBootstrapServices()`方法中，启动并注册 ActivityManagerService。

### 使用 SystemServiceManager 启动 ActivityManagerService 实例

SystemServiceManager 用来管理 SystemService 的生命周期，如创建、启动等。在`startBootstrapServices()`方法中调用`mSystemServiceManager`对象的`startService()`方法，创建并启动 ActivityManagerService 服务。此方法传入的参数是一个 ActivityManagerService.Lifecycle 的静态嵌套类对象：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    // ...

    public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
    }

    // ...

}
```

它继承了 SystemService，维护了 ActivityManagerService 的实例`mService`。

`startService()`采用反射的方式，创建 ActivityManagerService.Lifecycle 的实例，回调其`onStart()`方法将 ActivityManagerService 启动，并将此实例返回：

```java
// frameworks/base/services/core/java/com/android/server/SystemServiceManager.java

/**
 * Manages creating, starting, and other lifecycle events of
 * {@link com.android.server.SystemService system services}.
 *
 * {@hide}
 */
public class SystemServiceManager {
    // ...

    // Services that should receive lifecycle events.
    private final ArrayList<SystemService> mServices = new ArrayList<SystemService>();

    // ...

    /**
     * Creates and starts a system service. The class must be a subclass of
     * {@link com.android.server.SystemService}.
     *
     * @param serviceClass A Java class that implements the SystemService interface.
     * @return The service instance, never null.
     * @throws RuntimeException if the service fails to start.
     */
    @SuppressWarnings("unchecked")
    public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {

            // ...

            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext);
            }

            // ...

            // Register it.
            mServices.add(service);

            // Start it.
            try {
                service.onStart();
            } catch (RuntimeException ex) {
                throw new RuntimeException("Failed to start service " + name
                        + ": onStart threw an exception", ex);
            }
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }
}
```

ActivityManagerService.Lifecycle 实例的`getService()`方法就会返回 ActivityManagerService 实例。

### 在 ServiceManager 中注册 ActivityManagerService

在 SystemServer 的`startBootstrapServices()`方法中，已经创建并启动了 ActivityManagerService 实例`mActivityManagerService`。现调用`mActivityManagerService.setSystemProcess()`，在 ServiceManager 中注册此实例以及其它服务：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

/**
 * Use with {@link #getSystemService} to retrieve a
 * {@link android.app.ActivityManager} for interacting with the global
 * system state.
 *
 * @see #getSystemService
 * @see android.app.ActivityManager
 */
public static final String ACTIVITY_SERVICE = "activity";

// ...

public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);

        // ...

    }

    // ...

}
```

ServiceManager 用于注册各种系统服务，包括 ActivityManagerService，这些服务都是 Binder 对象，可用于进程间通信：

```java
// frameworks/base/core/java/android/os/ServiceManager.java

/** @hide */
public final class ServiceManager {
    private static final String TAG = "ServiceManager";

    private static IServiceManager sServiceManager;
    private static HashMap<String, IBinder> sCache = new HashMap<String, IBinder>();

    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
        return sServiceManager;
    }

    /**
     * Returns a reference to a service with the given name.
     * 
     * @param name the name of the service to get
     * @return a reference to the service, or <code>null</code> if the service doesn't exist
     */
    public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return getIServiceManager().getService(name);
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }

    /**
     * Place a new @a service called @a name into the service
     * manager.
     * 
     * @param name the name of the new service
     * @param service the service object
     */
    public static void addService(String name, IBinder service) {
        try {
            getIServiceManager().addService(name, service, false);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }

    /**
     * Place a new @a service called @a name into the service
     * manager.
     * 
     * @param name the name of the new service
     * @param service the service object
     * @param allowIsolated set to true to allow isolated sandboxed processes
     * to access this service
     */
    public static void addService(String name, IBinder service, boolean allowIsolated) {
        try {
            getIServiceManager().addService(name, service, allowIsolated);
        } catch (RemoteException e) {
            Log.e(TAG, "error in addService", e);
        }
    }
}
```

`getIServiceManager()`方法中，`BinderInternal.getContextObject()`是 native 方法，返回的是一个 ServiceManagerProxy 对象，是 IServiceManager 的一个客户端代理实现。这就得到了单例`sServiceManager`，通过它的`addService()`方法和`getService()`方法，可以向系统唯一的服务端注册和获取各类服务。

至此，ActivityManagerService 就创建、启动并注册完毕了。

回顾之前`ActivityManagerNative.getDefault()`方法，调用`ServiceManager.getService("activity")`显然就是获取 ActivityManagerService 实例，然后再调用`asInterface()`方法，转换为客户端代理 ActivityManagerProxy 对象并将其返回。可参考`frameworks/base/core/java/android/app/ActivityManagerNative.java`。

## ActivityManagerService#startActivity()

接着执行`Instrumentation#execStartActivity()`方法到：

```java
ActivityManagerNative.getDefault()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target != null ? target.mEmbeddedID : null,
                    requestCode, 0, null, options);
```

这里采用 IPC 机制，最终调用的是服务端的`ActivityManagerService#startActivity()`方法：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
            resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}

// ...

@Override
public final int startActivityAsUser(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
    enforceNotIsolatedCaller("startActivity");
    userId = mUserController.handleIncomingUser(Binder.getCallingPid(), Binder.getCallingUid(),
            userId, false, ALLOW_FULL_ONLY, "startActivity", null);
    // TODO: Switch to user app stacks here.
    return mActivityStarter.startActivityMayWait(caller, -1, callingPackage, intent,
            resolvedType, null, null, resultTo, resultWho, requestCode, startFlags,
            profilerInfo, null, null, bOptions, false, userId, null, null);
}
```

### ActivityStarter

`mActivityStarter`是一个 ActivityStarter 对象，ActivityStarter 是用来解释如何启动 Activity 的控制器类。转到`ActivityStarter#startActivityMayWait()`方法：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java

final int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, Intent intent, String resolvedType,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, IActivityManager.WaitResult outResult, Configuration config,
        Bundle bOptions, boolean ignoreTargetSecurity, int userId,
        IActivityContainer iContainer, TaskRecord inTask) {

    // ...

    synchronized (mService) {

        // ...

        int res = startActivityLocked(caller, intent, ephemeralIntent, resolvedType,
                aInfo, rInfo, voiceSession, voiceInteractor,
                resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags,
                options, ignoreTargetSecurity, componentSpecified, outRecord, container,
                inTask);

        // ...

        return res;
    }
}
```

转到`ActivityStarter#startActivityLocked()`方法，构造 ActivityRecord 信息并传递给`ActivityStarter#startActivityUnchecked()`方法：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java

final int startActivityLocked(IApplicationThread caller, Intent intent, Intent ephemeralIntent,
        String resolvedType, ActivityInfo aInfo, ResolveInfo rInfo,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        IBinder resultTo, String resultWho, int requestCode, int callingPid, int callingUid,
        String callingPackage, int realCallingPid, int realCallingUid, int startFlags,
        ActivityOptions options, boolean ignoreTargetSecurity, boolean componentSpecified,
        ActivityRecord[] outActivity, ActivityStackSupervisor.ActivityContainer container,
        TaskRecord inTask) {

    // ...

    try {
        mService.mWindowManager.deferSurfaceLayout();
        err = startActivityUnchecked(r, sourceRecord, voiceSession, voiceInteractor, startFlags,
                true, options, inTask);
    }

    // ...

}

// ...

private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask) {

    // ...

    // If the activity being launched is the same as the one currently at the top, then
    // we need to check if it should only be launched once.
    final ActivityStack topStack = mSupervisor.mFocusedStack;
    final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
    final boolean dontStart = top != null && mStartActivity.resultTo == null
            && top.realActivity.equals(mStartActivity.realActivity)
            && top.userId == mStartActivity.userId
            && top.app != null && top.app.thread != null
            && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
            || mLaunchSingleTop || mLaunchSingleTask);
    if (dontStart) {

        // ...

        if (mDoResume) {
            mSupervisor.resumeFocusedStackTopActivityLocked();
        }

        // ...

    }

    // ...

    mTargetStack.startActivityLocked(mStartActivity, newTask, mKeepCurTransition, mOptions);
    if (mDoResume) {
        if (!mLaunchTaskBehind) {
            // TODO(b/26381750): Remove this code after verification that all the decision
            // points above moved targetStack to the front which will also set the focus
            // activity.
            mService.setFocusedActivityLocked(mStartActivity, "startedActivity");
        }
        final ActivityRecord topTaskActivity = mStartActivity.task.topRunningActivityLocked();
        if (!mTargetStack.isFocusable()
                || (topTaskActivity != null && topTaskActivity.mTaskOverlay
                && mStartActivity != topTaskActivity)) {

            // ...

        } else {
            mSupervisor.resumeFocusedStackTopActivityLocked(mTargetStack, mStartActivity,
                    mOptions);
        }
    } else {
        mTargetStack.addRecentActivityLocked(mStartActivity);
    }

    // ...

    return START_SUCCESS;
}
```

注意到`dontStart`这个 boolean 值，它指示即将启动的 Activity 是否：

- 与当前栈顶的 Activity 相同
- 以 SingleTop 或 SingleTask 模式启动

如果同时满足以上两个条件，栈顶 Activity 会进行复用，显然不应该回调`onStart()`方法。

如果`mDoResume`为真，进行`onResume()`回调：

```
mSupervisor.resumeFocusedStackTopActivityLocked();
```

### ActivityStackSupervisor 与 ActivityStack

`mSupervisor`是一个 ActivityStackSupervisor 对象。ActivityStackSupervisor 用于管理 Activity 栈，ActivityStack 用于管理保存在栈中的 Activity。接着是 ActivityStackSupervisor 和 ActivityStack 的一系列调用：

```java
// frameworks/base/services/core/java/com/android/server/am/ActivityStackSupervisor.java

boolean resumeFocusedStackTopActivityLocked() {
    return resumeFocusedStackTopActivityLocked(null, null, null);
}

boolean resumeFocusedStackTopActivityLocked(
        ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {
    if (targetStack != null && isFocusedStack(targetStack)) {
        return targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
    }
    final ActivityRecord r = mFocusedStack.topRunningActivityLocked();
    if (r == null || r.state != RESUMED) {
        mFocusedStack.resumeTopActivityUncheckedLocked(null, null);
    }
    return false;
}
```


