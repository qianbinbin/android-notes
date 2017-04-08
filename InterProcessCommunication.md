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

Android 系统中已经有很多实现了 Parcelable 接口的类，例如 Intent、Bundle、Bitmap 等。List 和 Map 也可以序列化，只要它们里面的每个元素都是可序列化的。
Parcelable 和 Serializable 对比：
Serializable 使用简单，但开销很大，序列化和反序列化过程需要大量 I/O 操作；Parcelable 使用稍微麻烦一些，但效率高，也是 Android 推荐的序列化方式，因此首选 Parcelable。内存序列化，使用 Parcelable；持久化数据或网络传输，建议使用 Serializable。
