# IPC 机制
## Android 中的多进程
### 开启多进程
Android 中开启多进程，常规方法只有一种，即为四大组件（Activity、Service、Receiver、ContentProvider）在 AndroidManifest.xml 文件中指定`android:process`属性，无法为一个线程或实体类指定其运行时所在的进程。还有一种非常规方法，通过 JNI 在 native 层 fork 一个新的进程。
如果不指定 process 属性，则运行在默认进程中，默认进程的进程名是包名。
在指定 process 属性时，如果进程名以`:`开头，则默认为进程名前加上当前包名，表示此进程属于当前应用私有进程，其他应用的组件不能与其在同一进程中；如果进程名不以`:`开头，则此进程属于全局进程，其他应用通过 ShareUID 方式可以与其跑在同一进程中。
Android 系统会为每个应用分配一个唯一的 UID，具有相同 UID 的应用才能共享数据。不同应用可以使用相同的 ShareUID 让两个应用使用相同的 UID，从而共享私有数据，如 data 目录、组件信息等。如果使用了相同的 ShareUID，且签名相同，则它们跑在同一进程中，还可以共享内存数据，看起来就像是一个应用的两个部分。
### 多进程带来的问题
不同的进程，不能直接共享内存数据。
Android 为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，访问同一个类的对象会产生多份副本。
多进程带来的问题包括但不限于：
- 无法共享内存数据，例如静态成员、单例模式、线程同步机制完全失效。
- 读写文件同步问题，不同的进程在并发读/写、写/写同一文件时，所得结果可能会与期望结果不同，如SharedPreference、数据库等可靠性下降。
- Application 会重复创建，因为不同进程中的组件是属于不同的虚拟机和 Application 的。

## IPC 基础概念
### Serializable 接口
Serializable 是 Java 提供的一个序列化接口。要让一个对象实现序列化，只需这个类实现 Serializable 接口。序列化和反序列化过程几乎都被系统自动完成了。
- 序列化过程使用 ObjectOutputStream，将一个 OutputStream 对象传递给其构造方法，然后调用其 writeObject 方法。
- 反序列化过程使用 ObjectInputStream，将一个 InputStream 对象传递给其构造方法，然后调用其 readObject 方法。
Java 序列化只支持字节流（OutputStream、InputStream 的子类），不支持字符流（Writer、Reader 的子类）。以 Stream 结尾的类都是字节流，以 Writer 或 Reader 结尾的类都是字符流。

serialVersionUID 的作用是辅助序列化和反序列化过程，原则上序列化后的数据中的 serialVersionUID，只有和当前类的 serialVersionUID 相同时，才能反序列化。在序列化时，系统会把当前类的 serialVersionUID 写入序列化数据中，反序列化时系统会检测数据中的 serialVersionUID，看是否与当前类的 serialVersionUID 相同，如果相同，则可以反序列化，反之则不可以。
- 如果我们手动在待序列化类中指定 serialVersionUID 的值，例如：
```private static final long serialVersionUID = 1L;```
或者让 IDE 生成 serialVersionUID，则在类修改之后，例如新增或删除了成员变量，反序列化仍然能够成功，程序会最大限度地恢复数据。
但当类结构发生了非常规性改变，例如修改了类名，修改了成员变量类型等，即使 serialVersionUID 相同，反序列化还是会失败。
- 如果不手动指定 serialVersionUID，则 Java 编译器会计算类的 serialVersionUID，其值依赖于编译器实现。在修改此类后，serialVersionUID 值将会改变，反序列化失败。

值得注意的是，静态成员变量属于类，不属于对象，因此不会参与序列化过程。另外，使用 transient 关键字修饰的成员也不参与序列化过程。
在需要序列化的类中定义 writeObject 和 readObject 方法，可以修改默认的序列化和反序列化过程，虚拟机会先试图调用类中的这两个方法，如果不存在，则默认调用是 ObjectOutputStream 的 defaultWriteObject 方法以及 ObjectInputStream 的 defaultReadObject 方法。
### Parcelable 接口
Parcelable 是 Android 提供的一个接口，一个类只要实现这个接口，它的对象就可以实现序列化并可以通过 Intent 和 Binder 传递。
- 序列化：重写 writeToParcel 方法
- 反序列化：根据 IDE 提示，创建一个 Parcelable.Creator<T> 的匿名内部类的 CREATOR 对象，并实现 createFromParcel 和 newArray 方法。

需要注意的是，writeToParcel 和 createFromParcel 方法对数据成员的读写顺序要保持一致。
Android 系统中已经有很多实现了 Parcelable 接口的类，例如 Intent、Bundle、Bitmap 等。List 和 Map 也可以序列化，只要它们里面的每个元素都是可序列化的。
Parcelable 和 Serializable 对比：
Serializable 使用简单，但开销很大，序列化和反序列化过程需要大量 I/O 操作；Parcelable 使用稍微麻烦一些，但效率高，也是 Android 推荐的序列化方式，因此首选 Parcelable。一般来说，内存序列化，使用 Parcelable；持久化数据或网络传输，建议使用 Serializable。
### Binder
用户空间的各应用程序，通过内核空间的 Binder 驱动进行通信，Binder 相当于一个中转站。
- Binder 是 Android 中的一个类，它实现了 IBinder 接口，是 Android 中的一种跨进程通信方式。
- Binder 还可以理解为一种虚拟的物理设备，设备驱动为 /dev/binder。
- Binder 是 ServiceManager 连接各种 Manager（ActivityManager、WindowManager 等）和相应 ManagerService 的桥梁。
- Binder 是客户端和服务端通信的媒介，当 bindService 的时候，服务端会返回一个包含服务端业务调用的 Binder 对象，客户端以此获取服务端提供的服务或数据。这里的服务包括普通服务和基于 AIDL 的服务。

Binder 主要用在 Service 中，包括 AIDL 和 Messenger。其中普通 Service 中的 Binder 不涉及进程间通信，Messenger 的底层其实是 AIDL。
AIDL（Android 接口定义语言）支持的数据类型：
- Java 编程语言中的所有原语类型（如 int、long、char、boolean 等等）
- String 和 CharSequence
- List：只支持 ArrayList，且其中元素都必须被 AIDL 支持
- Map：只支持 HashMap，且其中元素都必须被 AIDL 支持
- 实现 Parcelable 接口的对象
- 所有 AIDL 接口

下面以 Android Studio 为开发环境，用一个实例说明 AIDL 的简单使用方法：
1. 在 Android Studio 中调整视图为 Android 模式，java 文件夹和 aidl 文件夹（添加 AIDL 后自动创建）应处于同级目录。
2. 要在 AIDL 中使用自定义类，例如 Book 类，该类必须实现 Parcelable 接口，`Book.java`文件显然保存在 java 目录下。
3. 为了在 AIDL 中使用 Book 类，还要新建其对应的 AIDL 文件`Book.aidl`，其包名和 Book 类相同。Book.aidl 文件就在 aidl 相应的目录下，目录结构与 Book.java 相同。在其中声明 Book 类，以供其它 AIDL 使用：
```parcelable Book;```
parcelable 是一种类型，注意与 Parcelable 接口区分。
4. 定义所需接口文件`IBookManager.aidl`，AIDL 接口使用 Java 语言构建。
 - 需要注意不管 Book 类是否与其处于同一包中，都要 import 这个类。定义的 .aidl 文件就是服务端提供的接口，供客户端调用，因此要考虑兼容性（见第 5 条 onTransact 方法说明）。
5. SDK 工具会在`app/build/generated/source/aidl/debug`下自动生成对应的 Java 接口文件`IBookManager.java`，调整视图为 Project 可以看到。其结构分析如下：
  - IBookManager 接口继承了 IInterface 接口，定义 Binder 接口时都必须继承 IInterface 接口。接口中声明了刚才在 AIDL 接口中声明的方法。它还有一个继承了 Binder 类的内部类 Stub，Stub 内部有一个代理类 Proxy。
    - DESCRIPTOR：Binder 的唯一标识，一般就是当前 Binder 的类名。
    - asInterface：将服务端的 Binder 对象转换为客户端所需的 AIDL 接口类型的对象。如果客户端和服务端在同一进程中，返回的就是服务端 Stub 对象本身，否则返回的是系统封装后的 Stub.Proxy 对象，查阅 Binder 源码可知，就是通过 DESCRIPTOR 来判断的。
    - asBinder：返回当前 Binder 对象。
    - onTransact：参数为 code、data、reply、flags。code 即为 transaction code，服务端以此可以确定客户端请求的目标方法是什么，接着从 data 中取出目标方法所需参数（如果目标方法有参数的话），然后执行目标方法。目标方法执行完毕后，向 reply 中写入返回值（如果目标方法有返回值的话）。需要注意的是，如果 onTransact 方法返回 false，那么客户端请求会失败，我们可以利用这个特性做权限验证。可以看到，code 实际上对应的是目标方法的标识，且随着方法声明的顺序逐项增加。考虑到兼容性问题，服务端对 AIDL 接口中方法的修改就必须：
      1. 新增方法时只能在最后添加，不能在中间添加
      2. 不能删除方法
      3. 不能改变方法声明顺序
      在客户端，若请求的方法不存在，并不会发生异常。如果该方法有返回类值，则返回它的默认值。
    - Proxy：此类的对象中的方法允许客户端调用，其内部实现为：
      1. 创建该方法所需输入型 Parcel 对象 _data、输出型 Parcel 对象 _reply、返回值对象（如果有的话），并把参数写入 _data（如果有参数的话）
      2. 调用 transact 方法来发起 RPC（Remote Procedure Call），当前线程挂起
      3. 服务端 onTransact 调用，RPC 过程返回后，客户端线程继续执行，并从 _reply 中取出 RPC 过程的返回结果，最后返回 _reply 中的数据
6. 服务端实现 IBookManager 接口，例如在自定义 Service 中实例化一个 IBookManager.Stub 对象 mBinder，并实现声明的方法，并在 onBind 方法中返回。
7. 把服务端的 AIDL 文件（`IBookManager.aidl`、`Book.aidl`），以及所需的实现 Parcelable 接口的自定义类（`Book.java`）都拷贝到客户端，以便调用。
8. 客户端实现 ServiceConnection 接口并产生一个实例，如 mConnection，调用 bindService 连接此服务，客户端的 onServiceConnected 方法就会接收到服务端的 onBind 方法返回的 mBinder 实例。

值得注意的是，
1. 隐式调用 Service 时，需要同时设置 Service 所属包名，例如：
```
Intent intent = new Intent();
intent.setAction("com.example.YOURACTION");
intent.setPackageName("com.example.packageName");
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```
2. 使用`adb shell dumpsys activity services`可以查看当前启动的 Service 和连接，经实验可知：
  - 如果 Service 以 startService 方法启动过任何一次，则只有在没有任何客户端与此 Service 绑定、且调用了 stopService 方法的情况下，Service 才会停止，两个条件缺一不可。
  - 如果 Service 仅仅以 bindService 方式启动，在所有客户端都解除绑定后，Service 自动 onDestroy。

这与各方面预期相同。以 startService 启动的一方，肯定不希望 Service 莫名其妙 onDestroy。仅以 bindService 方式开启的 Service，在解除所有连接后，自然希望 Service 自动销毁。

显然，供客户端调用的 Proxy 中方法的过程是：先将参数序列化，然后发起 RPC，最后反序列化得到返回值；服务端 onTransact 中的过程与之相反：先将客户端传来的参数反序列化，然后执行目标方法，最后将客户端所需结果序列化。
需要注意的是，客户端发起请求后，当前线程会被挂起直至服务端返回数据，如果一个远程方法比较耗时，则客户端请求应在后台线程发起；而服务端 Binder 方法运行在 Binder 线程池中，考虑到线程安全，不管其是否耗时，都应采用同步的方式去实现。

事实上，我们完全可以不使用 AIDL 文件而直接实现 Binder，AIDL 文件的作用只是让系统生成代码，主要要修改的有：
1. 在 Java 代码中声明 IBookManager 接口并继承 IInterface 接口，仿照 AIDL 生成的代码，声明所需方法，建议同时声明对应的 transaction code 以及 DESCRIPTOR 标识，而不是像自动生成的代码那样，把 transaction code 和 DESCRIPTOR 声明在 Stub 类中。这样便于维护，因为 transaction code 必须与方法相关。
2. 仿照原先的 Stub 类，继承 Binder 并实现 IBookManager 接口，作为服务端类，例如：
```
public class BookManagerImpl extends Binder implements IBookManager {}
```
其内容与原先的 Stub 类相同。
其中的 Proxy 类也可以单独拿出来，使得代码结构更加清晰，其内容与原先相同。
如果想把一些方法放到其它地方实现，就像原先在自定义 Service 中的 mBinder 对象那样，可以将 BookManagerImpl 声明为 abstract 类，然后在需要实现的地方继承它。
当然这些都还要复制到客户端，其它地方稍作改动即可。由此看来，完全可以不使用 AIDL 文件。
