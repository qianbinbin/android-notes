# IPC ����
## Android �еĶ����
### ���������
Android �п�������̣����淽��ֻ��һ�֣���Ϊ�Ĵ������Activity��Service��Receiver��ContentProvider���� AndroidManifest.xml �ļ���ָ��`android:process`���ԣ��޷�Ϊһ���̻߳�ʵ����ָ��������ʱ���ڵĽ��̡�����һ�ַǳ��淽����ͨ�� JNI �� native �� fork һ���µĽ��̡�
�����ָ�� process ���ԣ���������Ĭ�Ͻ����У�Ĭ�Ͻ��̵Ľ������ǰ�����
��ָ�� process ����ʱ�������������`:`��ͷ����Ĭ��Ϊ������ǰ���ϵ�ǰ��������ʾ�˽������ڵ�ǰӦ��˽�н��̣�����Ӧ�õ��������������ͬһ�����У��������������`:`��ͷ����˽�������ȫ�ֽ��̣�����Ӧ��ͨ�� ShareUID ��ʽ������������ͬһ�����С�
Android ϵͳ��Ϊÿ��Ӧ�÷���һ��Ψһ�� UID��������ͬ UID ��Ӧ�ò��ܹ������ݡ���ͬӦ�ÿ���ʹ����ͬ�� ShareUID ������Ӧ��ʹ����ͬ�� UID���Ӷ�����˽�����ݣ��� data Ŀ¼�������Ϣ�ȡ����ʹ������ͬ�� ShareUID����ǩ����ͬ������������ͬһ�����У������Թ����ڴ����ݣ�������������һ��Ӧ�õ��������֡�
### ����̴���������
��ͬ�Ľ��̣�����ֱ�ӹ����ڴ����ݡ�
Android Ϊÿ�����̶�����һ�����������������ͬ����������ڴ�������в�ͬ�ĵ�ַ�ռ䣬����ͬһ����Ķ���������ݸ�����
����̴�������������������ڣ�
- �޷������ڴ����ݣ����羲̬��Ա������ģʽ���߳�ͬ��������ȫʧЧ��
- ��д�ļ�ͬ�����⣬��ͬ�Ľ����ڲ�����/д��д/дͬһ�ļ�ʱ�����ý�����ܻ������������ͬ����SharedPreference�����ݿ�ȿɿ����½���
- Application ���ظ���������Ϊ��ͬ�����е���������ڲ�ͬ��������� Application �ġ�

## IPC ��������
### Serializable �ӿ�
Serializable �� Java �ṩ��һ�����л��ӿڡ�Ҫ��һ������ʵ�����л���ֻ�������ʵ�� Serializable �ӿڡ����л��ͷ����л����̼�������ϵͳ�Զ�����ˡ�
- ���л�����ʹ�� ObjectOutputStream����һ�� OutputStream ���󴫵ݸ��乹�췽����Ȼ������� writeObject ������
- �����л�����ʹ�� ObjectInputStream����һ�� InputStream ���󴫵ݸ��乹�췽����Ȼ������� readObject ������
Java ���л�ֻ֧���ֽ�����OutputStream��InputStream �����ࣩ����֧���ַ�����Writer��Reader �����ࣩ���� Stream ��β���඼���ֽ������� Writer �� Reader ��β���඼���ַ�����

serialVersionUID �������Ǹ������л��ͷ����л����̣�ԭ�������л���������е� serialVersionUID��ֻ�к͵�ǰ��� serialVersionUID ��ͬʱ�����ܷ����л��������л�ʱ��ϵͳ��ѵ�ǰ��� serialVersionUID д�����л������У������л�ʱϵͳ���������е� serialVersionUID�����Ƿ��뵱ǰ��� serialVersionUID ��ͬ�������ͬ������Է����л�����֮�򲻿��ԡ�
- ��������ֶ��ڴ����л�����ָ�� serialVersionUID ��ֵ�����磺
```private static final long serialVersionUID = 1L;```
������ IDE ���� serialVersionUID���������޸�֮������������ɾ���˳�Ա�����������л���Ȼ�ܹ��ɹ������������޶ȵػָ����ݡ�
������ṹ�����˷ǳ����Ըı䣬�����޸����������޸��˳�Ա�������͵ȣ���ʹ serialVersionUID ��ͬ�������л����ǻ�ʧ�ܡ�
- ������ֶ�ָ�� serialVersionUID���� Java �������������� serialVersionUID����ֵ�����ڱ�����ʵ�֡����޸Ĵ����serialVersionUID ֵ����ı䣬�����л�ʧ�ܡ�

ֵ��ע����ǣ���̬��Ա���������࣬�����ڶ�����˲���������л����̡����⣬ʹ�� transient �ؼ������εĳ�ԱҲ���������л����̡�
����Ҫ���л������ж��� writeObject �� readObject �����������޸�Ĭ�ϵ����л��ͷ����л����̣������������ͼ�������е���������������������ڣ���Ĭ�ϵ����� ObjectOutputStream �� defaultWriteObject �����Լ� ObjectInputStream �� defaultReadObject ������
### Parcelable �ӿ�
Parcelable �� Android �ṩ��һ���ӿڣ�һ����ֻҪʵ������ӿڣ����Ķ���Ϳ���ʵ�����л�������ͨ�� Intent �� Binder ���ݡ�
- ���л�����д writeToParcel ����
- �����л������� IDE ��ʾ������һ�� Parcelable.Creator<T> �������ڲ���� CREATOR ���󣬲�ʵ�� createFromParcel �� newArray ������

��Ҫע����ǣ�writeToParcel �� createFromParcel ���������ݳ�Ա�Ķ�д˳��Ҫ����һ�¡�
Android ϵͳ���Ѿ��кܶ�ʵ���� Parcelable �ӿڵ��࣬���� Intent��Bundle��Bitmap �ȡ�List �� Map Ҳ�������л���ֻҪ���������ÿ��Ԫ�ض��ǿ����л��ġ�
Parcelable �� Serializable �Աȣ�
Serializable ʹ�ü򵥣��������ܴ����л��ͷ����л�������Ҫ���� I/O ������Parcelable ʹ����΢�鷳һЩ����Ч�ʸߣ�Ҳ�� Android �Ƽ������л���ʽ�������ѡ Parcelable��һ����˵���ڴ����л���ʹ�� Parcelable���־û����ݻ����紫�䣬����ʹ�� Serializable��
### Binder
�û��ռ�ĸ�Ӧ�ó���ͨ���ں˿ռ�� Binder ��������ͨ�ţ�Binder �൱��һ����תվ��
- Binder �� Android �е�һ���࣬��ʵ���� IBinder �ӿڣ��� Android �е�һ�ֿ����ͨ�ŷ�ʽ��
- Binder ���������Ϊһ������������豸���豸����Ϊ /dev/binder��
- Binder �� ServiceManager ���Ӹ��� Manager��ActivityManager��WindowManager �ȣ�����Ӧ ManagerService ��������
- Binder �ǿͻ��˺ͷ����ͨ�ŵ�ý�飬�� bindService ��ʱ�򣬷���˻᷵��һ�����������ҵ����õ� Binder ���󣬿ͻ����Դ˻�ȡ������ṩ�ķ�������ݡ�����ķ��������ͨ����ͻ��� AIDL �ķ���

Binder ��Ҫ���� Service �У����� AIDL �� Messenger��������ͨ Service �е� Binder ���漰���̼�ͨ�ţ�Messenger �ĵײ���ʵ�� AIDL��
AIDL��Android �ӿڶ������ԣ�֧�ֵ��������ͣ�
- Java ��������е�����ԭ�����ͣ��� int��long��char��boolean �ȵȣ�
- String �� CharSequence
- List��ֻ֧�� ArrayList��������Ԫ�ض����뱻 AIDL ֧��
- Map��ֻ֧�� HashMap��������Ԫ�ض����뱻 AIDL ֧��
- ʵ�� Parcelable �ӿڵĶ���
- ���� AIDL �ӿ�

������ Android Studio Ϊ������������һ��ʵ��˵�� AIDL �ļ�ʹ�÷�����
1. �� Android Studio �е�����ͼΪ Android ģʽ��java �ļ��к� aidl �ļ��У���� AIDL ���Զ�������Ӧ����ͬ��Ŀ¼��
2. Ҫ�� AIDL ��ʹ���Զ����࣬���� Book �࣬�������ʵ�� Parcelable �ӿڣ�`Book.java`�ļ���Ȼ������ java Ŀ¼�¡�
3. Ϊ���� AIDL ��ʹ�� Book �࣬��Ҫ�½����Ӧ�� AIDL �ļ�`Book.aidl`��������� Book ����ͬ��Book.aidl �ļ����� aidl ��Ӧ��Ŀ¼�£�Ŀ¼�ṹ�� Book.java ��ͬ������������ Book �࣬�Թ����� AIDL ʹ�ã�
```parcelable Book;```
parcelable ��һ�����ͣ�ע���� Parcelable �ӿ����֡�
4. ��������ӿ��ļ�`IBookManager.aidl`��AIDL �ӿ�ʹ�� Java ���Թ�����
 - ��Ҫע�ⲻ�� Book ���Ƿ����䴦��ͬһ���У���Ҫ import ����ࡣ����� .aidl �ļ����Ƿ�����ṩ�Ľӿڣ����ͻ��˵��ã����Ҫ���Ǽ����ԣ����� 5 �� onTransact ����˵������
5. SDK ���߻���`app/build/generated/source/aidl/debug`���Զ����ɶ�Ӧ�� Java �ӿ��ļ�`IBookManager.java`��������ͼΪ Project ���Կ�������ṹ�������£�
  - IBookManager �ӿڼ̳��� IInterface �ӿڣ����� Binder �ӿ�ʱ������̳� IInterface �ӿڡ��ӿ��������˸ղ��� AIDL �ӿ��������ķ�����������һ���̳��� Binder ����ڲ��� Stub��Stub �ڲ���һ�������� Proxy��
    - DESCRIPTOR��Binder ��Ψһ��ʶ��һ����ǵ�ǰ Binder ��������
    - asInterface��������˵� Binder ����ת��Ϊ�ͻ�������� AIDL �ӿ����͵Ķ�������ͻ��˺ͷ������ͬһ�����У����صľ��Ƿ���� Stub ���������򷵻ص���ϵͳ��װ��� Stub.Proxy ���󣬲��� Binder Դ���֪������ͨ�� DESCRIPTOR ���жϵġ�
    - asBinder�����ص�ǰ Binder ����
    - onTransact������Ϊ code��data��reply��flags��code ��Ϊ transaction code��������Դ˿���ȷ���ͻ��������Ŀ�귽����ʲô�����Ŵ� data ��ȡ��Ŀ�귽��������������Ŀ�귽���в����Ļ�����Ȼ��ִ��Ŀ�귽����Ŀ�귽��ִ����Ϻ��� reply ��д�뷵��ֵ�����Ŀ�귽���з���ֵ�Ļ�������Ҫע����ǣ���� onTransact �������� false����ô�ͻ��������ʧ�ܣ����ǿ����������������Ȩ����֤�����Կ�����code ʵ���϶�Ӧ����Ŀ�귽���ı�ʶ�������ŷ���������˳���������ӡ����ǵ����������⣬����˶� AIDL �ӿ��з������޸ľͱ��룺
      1. ��������ʱֻ���������ӣ��������м����
      2. ����ɾ������
      3. ���ܸı䷽������˳��
      �ڿͻ��ˣ�������ķ��������ڣ������ᷢ���쳣������÷����з�����ֵ���򷵻�����Ĭ��ֵ��
    - Proxy������Ķ����еķ�������ͻ��˵��ã����ڲ�ʵ��Ϊ��
      1. �����÷������������� Parcel ���� _data������� Parcel ���� _reply������ֵ��������еĻ��������Ѳ���д�� _data������в����Ļ���
      2. ���� transact ���������� RPC��Remote Procedure Call������ǰ�̹߳���
      3. ����� onTransact ���ã�RPC ���̷��غ󣬿ͻ����̼߳���ִ�У����� _reply ��ȡ�� RPC ���̵ķ��ؽ������󷵻� _reply �е�����
6. �����ʵ�� IBookManager �ӿڣ��������Զ��� Service ��ʵ����һ�� IBookManager.Stub ���� mBinder����ʵ�������ķ��������� onBind �����з��ء�
7. �ѷ���˵� AIDL �ļ���`IBookManager.aidl`��`Book.aidl`�����Լ������ʵ�� Parcelable �ӿڵ��Զ����ࣨ`Book.java`�����������ͻ��ˣ��Ա���á�
8. �ͻ���ʵ�� ServiceConnection �ӿڲ�����һ��ʵ������ mConnection������ bindService ���Ӵ˷��񣬿ͻ��˵� onServiceConnected �����ͻ���յ�����˵� onBind �������ص� mBinder ʵ����

ֵ��ע����ǣ�
1. ��ʽ���� Service ʱ����Ҫͬʱ���� Service �������������磺
```
Intent intent = new Intent();
intent.setAction("com.example.YOURACTION");
intent.setPackageName("com.example.packageName");
bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
```
2. ʹ��`adb shell dumpsys activity services`���Բ鿴��ǰ������ Service �����ӣ���ʵ���֪��
  - ��� Service �� startService �����������κ�һ�Σ���ֻ����û���κοͻ������ Service �󶨡��ҵ����� stopService ����������£�Service �Ż�ֹͣ����������ȱһ���ɡ�
  - ��� Service ������ bindService ��ʽ�����������пͻ��˶�����󶨺�Service �Զ� onDestroy��

���������Ԥ����ͬ���� startService ������һ�����϶���ϣ�� Service Ī������ onDestroy������ bindService ��ʽ������ Service���ڽ���������Ӻ���Ȼϣ�� Service �Զ����١�

��Ȼ�����ͻ��˵��õ� Proxy �з����Ĺ����ǣ��Ƚ��������л���Ȼ���� RPC��������л��õ�����ֵ������� onTransact �еĹ�����֮�෴���Ƚ��ͻ��˴����Ĳ��������л���Ȼ��ִ��Ŀ�귽������󽫿ͻ������������л���
��Ҫע����ǣ��ͻ��˷�������󣬵�ǰ�̻߳ᱻ����ֱ������˷������ݣ����һ��Զ�̷����ȽϺ�ʱ����ͻ�������Ӧ�ں�̨�̷߳��𣻶������ Binder ���������� Binder �̳߳��У����ǵ��̰߳�ȫ���������Ƿ��ʱ����Ӧ����ͬ���ķ�ʽȥʵ�֡�

��ʵ�ϣ�������ȫ���Բ�ʹ�� AIDL �ļ���ֱ��ʵ�� Binder��AIDL �ļ�������ֻ����ϵͳ���ɴ��룬��ҪҪ�޸ĵ��У�
1. �� Java ���������� IBookManager �ӿڲ��̳� IInterface �ӿڣ����� AIDL ���ɵĴ��룬�������跽��������ͬʱ������Ӧ�� transaction code �Լ� DESCRIPTOR ��ʶ�����������Զ����ɵĴ����������� transaction code �� DESCRIPTOR ������ Stub ���С���������ά������Ϊ transaction code �����뷽����ء�
2. ����ԭ�ȵ� Stub �࣬�̳� Binder ��ʵ�� IBookManager �ӿڣ���Ϊ������࣬���磺
```
public class BookManagerImpl extends Binder implements IBookManager {}
```
��������ԭ�ȵ� Stub ����ͬ��
���е� Proxy ��Ҳ���Ե����ó�����ʹ�ô���ṹ������������������ԭ����ͬ��
������һЩ�����ŵ������ط�ʵ�֣�����ԭ�����Զ��� Service �е� mBinder �������������Խ� BookManagerImpl ����Ϊ abstract �࣬Ȼ������Ҫʵ�ֵĵط��̳�����
��Ȼ��Щ����Ҫ���Ƶ��ͻ��ˣ������ط������Ķ����ɡ��ɴ˿�������ȫ���Բ�ʹ�� AIDL �ļ���
