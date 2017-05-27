# Activity 启动原理

由于 Activity 启动的实现较为繁杂，本文将专注于 Activity 启动过程，避免纠结于细节。

通常启动一个 Activity 十分简单，例如从当前的 Activity 启动另一个 Activity，一个典型的显式 Intent 启动方式如下：

```
Intent intent = new Intent(this, TargetActivity.class);
startActivity(intent);
```

从`frameworks/base/core/java/android/app/Activity.java`源码看，启动 Activity 有很多种方法，实际上实现的原理大同小异，下面以`startActivity(Intent)`为例说明 Activity 的启动过程。

`startActivity(Intent)`最终调用的是`startActivityForResult(Intent, int, Bundle)`方法：

```java
// frameworks/base/core/java/android/app/Activity.java

public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
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

这里的`mInstrumentation`是 Instrumentation 类型的对象，这个类主要用于监听系统和应用的交互。

`mMainThread`是 ActivityThread 类型对象。ActivityThread 运行在主线程中，其`main()`方法是主线程的入口，因此常把 ActivityThread 称作主线程，`mMainThread`就代表了当前应用的主线程。

`mMainThread.getApplicationThread()`获取的是一个 ApplicationThread 类型的对象，ApplicationThread 是 ActivityThread 的一个内部类。而 ApplicationThread 继承了 ApplicationThreadNative，这是一个 Binder。显然，这里获取到的 ApplicationThread 对象是一个运行在主线程中的 Binder 服务，利用它来进行进程间通信。

`mToken`是一个 Binder 对象的远程接口。

```java
// frameworks/base/core/java/android/app/Instrumentation.java

public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {

    // ...

    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
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

抽象类 ActivityManagerNative 是一个 Binder，它实现了 IActivityManager 接口。静态方法`ActivityManagerNative.getDefault()`返回的是 IActivityManager 实例，它实际上使用了懒汉方式实现的单例模式：

```
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

}
```

ActivityManagerService 继承了 ActivityManagerNative，是 IActivityManager 的具体实现，作为服务端 Binder。
