# 《Android源码设计模式解析与实战》一书

## [面向对象的六大原则](../misc/OOP6Principles.md)

## 单例模式
+  建议实现方式
  +  无参数时，static inner holder class方式：
  
  ```java
  public class Singleton {
	  private Singleton() {
		  // singleton
	  }
	  
	  public static getInstance() {
		  return InstanceHolder.sInstance;
	  }
	  
	  private static class InstanceHolder {
		  private static final Singleton sInstance = new Singleton();
	  }
  }
  ```
  
  +  有参数时，优化的double check lock (DCL)方式，注意`volatile`关键字，在JDK1.5及以后，用于解决this指针逃逸问题：
  
  ```java
  public class Singleton {
      private static volatile Singleton sInstance;
    
      private final Context mContext;
    
	  private Singleton(Context context) {
		    mContext = context;
	  }
	  
	  public static getInstance(Context context) {
            if (sInstance == null) {
                synchronized(Singleton.class) {
                    if (sInstance == null) {
                        sInstance = new Singleton(context);
                    }
                }
            }
            
            return sInstance;
	  }
  }
  ```
  
  +  this指针逃逸问题
  
    在DCL单例实现中，`sInstance = new Singleton(context);`实际上大致会进行三个操作：
    
    1. 为`Singleton`的实例分配内存；
    2. 调用`Singleton`的构造函数，初始化成员变量；
    3. 把`sInstance`指向新分配的内存空间（此时`sInstance`就不是`null`了）；
    
    由于Java memory model的内存缓存模型（寄存器，cache，主存），以及允许指令重排，在缓存层面，上述三步中后两步顺序是无法保证的，有可能是`1-2-3`，也有可能是`1-3-2`。如果是`1-3-2`，在A线程执行，第三步执行完之后，切换到B线程，则B线程执行`getInstance`时将直接返回`sInstance`，因为B线程将直接命中缓存，但是它的成员变量仍未初始化，一旦使用就会出现问题。
    
    而`volatile`关键字则能保证每次读数据都从主存读取，不会受到缓存的影响，所以B线程从主存读到的仍是null，就不会出现问题了。
  
+  安卓系统的实践（部分）
  +  各种system service就是单例，虽然他们使用binder机制和真正的service跨进程通信，但是在本地的proxy就是单例的（APP进程中的单例）；
  +  每个APP都有各种各样的`Context`，它的实现类是`ComtextImpl`，它在static代码块中初始化各种system service，并保存在一个map中，后续获取的时候将直接从map中获取；android 23起，初始化system service、缓存、单例逻辑的代码从`ContextImpl`转移到了`SystemServiceRegistry`中；
  +  `LayoutInflater`工作原理
    +  实现类是`PhoneLayoutInflater`
    +  内置view，在xml定义时不用声明完整包名，因为`PhoneLayoutInflater`在`onCreateView`函数中会自动为其添加包名前缀
    +  最终view的创建都是通过`createView`函数完成，在其中通过反射创建view实例
    +  `inflate`函数会解析xml布局文件，深度优先遍历，逐层创建view（通过`createViewFromTag`函数），并最终形成一棵view的树
  
## Builder模式
+  [AutoValue](https://github.com/google/auto/tree/master/value)及其在Android平台的版本[AutoParcel](https://github.com/frankiesardo/auto-parcel)，不仅支持immutable，还支持builder模式
+  `WindowManager`
  +  实现类为`WindowManagerImpl`
  +  Activity, Fragment, Dialog等组建的view显示，都是通过的`Window::setContentView()`方法来实现的
  +  `Window`是抽象类，其实现类为`PhoneWindow`，**它的setContentView方法，怎么和WindowManager关联起来的？？**
  +  `WindowManager`添加、移除view的实现工作在`WindowManagerGlobal`类中
  +  `ViewRootImpl`是framework层与native层通信的桥梁，继承自`Handler`
  +  WMS只负责管理手机屏幕上View的z-order，即View的图层顺序，WMS管理的是属于某个window下的view

## 原型模式
+  在已有对象的基础上构造新对象，通常是clone；clone的效率通常比new的效率高，但不绝对；需要注意深浅拷贝的问题。
+  Intent的查找与匹配
  +  系统启动之后，`PackageManagerService`会扫描系统内所有的APP，解析其manifest文件，得到所有APP注册的所有各类组件，保存在系统中（内存）；后续使用Intent进行跳转时，通过查找保存的组件信息，得知应该启动哪个APP（组件）；
  +  显式Intent：直接指定了响应Activity；隐式Intent：只指定了Action；
  +  启动Activity时，会向PackageManagerService查询intent对应的activity，在`PackageManagerService::queryIntentActivities(...)`方法中；

## 工厂方法
+  用工厂去创建产品（对象），工厂和产品都可以提供抽象类，具体类，以达到实现解耦的目的
+  App的启动过程
  +  `ActivityThread::main()`是app的执行起点
  +  `thread.attach(false)`把ActivityThread绑定到ActivityManagerService
  +  `thread.attach(false)`调用了`IActivityManager::attachApplication`方法，其实现在`ActivityManagerService`中
  +  `IActivityManager::attachApplication`方法中间接调用了`IApplicationThread::bindApplication`和`ActivityManagerService::attachApplicationLocked`