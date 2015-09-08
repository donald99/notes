#Effective Java一书笔记

##对象的创建与销毁
+  Item 1: 使用static工厂方法，而不是构造函数创建对象  
仅仅是创建对象的方法，并非Factory Pattern
  +  优点
    +  命名、接口理解更高效，通过工厂方法的函数名，而不是参数列表来表达其语义
	+  Instance control，并非每次调用都会创建新对象，可以使用预先创建好的对象，或者做对象缓存；便于实现单例；或不可实例化的类；对于immutable的对象来说，使得用`==`判等符合语义，且更高效；
	+  工厂方法能够返回任何返回类型的子类对象，甚至是私有实现；使得开发模块之间通过接口耦合，降低耦合度；而接口的实现也将更加灵活；接口不能有static方法，通常做法是为其再创建一个工厂方法类，如Collection与Collections；
	+  Read More: Service Provider Framework
  +  缺点
    +  仅有static工厂方法，没有public/protected构造函数的类将无法被继承；见仁见智，这一方面也迫使开发者倾向于组合而非继承；
	+  Javadoc中不能和其他static方法区分开，没有构造函数的集中显示优点；但可以通过公约的命名规则来改善；
  +  小结  
  static工厂方法和public构造函数均有其优缺点，在编码过程中，可以先考虑一下工厂方法是否合适，再进行选择。
+  Item 2: 使用当构造函数的参数较多，尤其是其中还有部分是可选参数时，使用Builder模式
  +  以往的方法
    +  Telescoping constructor：针对可选参数，从0个到最多个，依次编写一个构造函数，它们按照参数数量由少到多逐层调用，最终调用到完整参数的构造函数；代码冗余，有时还得传递无意义参数，而且容易导致使用过程中出隐蔽的bug；
    +  JavaBeans Pattern：灵活，但是缺乏安全性，有状态不一致问题，线程安全问题；
  +  Builder Pattern
    +  代码灵活简洁；具备安全性；
    +  immutable
    +  参数检查：最好放在要build的对象的构造函数中，而非builder的构建过程中
    +  支持多个field以varargs的方式设置（每个函数只能有一个varargs）
    +  一个builder可以build多个对象
    +  Builder结合泛型，实现Abstract Factory Pattern
    +  传统的抽象工厂模式，是用Class类实现的，然而其有缺点：newInstance调用总是去调用无参数构造函数，不能保证存在；newInstance方法会抛出所有无参数构造函数中的异常，而且不会被编译期的异常检查机制覆盖；可能会导致运行时异常，而非编译期错误；
  +  小结  
  Builder模式在简单地类（参数较少，例如4个以下）中，优势并不明显，但是需要予以考虑，尤其是当参数可能会变多时，有可选参数时更是如此。
+  Item 3: 单例模式！  
不管以哪种形式实现单例模式，它们的核心原理都是将构造函数私有化，并且通过静态方法获取一个唯一的实例，在这个获取的过程中你必须保证线程安全、反序列化导致重新生成实例对象等问题，该模式简单，但使用率较高。
  +  double-check-locking  
    ```java
        private static volatile RestAdapter sRestAdapter = null;
        public static RestAdapter provideRestAdapter() {
            if (sRestAdapter == null) {
                synchronized (RestProvider.class) {
                    if (sRestAdapter == null) {
                        sRestAdapter = new RestAdapter();
                    }
                }
            }
    
            return sRestAdapter;
        }
    ```
  DCL可能会失效，因为指令重排可能导致同步解除后，对象初始化不完全就被其他线程获取；使用volatile关键字修饰对象，或者使用static SingletonHolder来避免该问题（后者JLS推荐）；
  +  class的static代码：一个类只有在被使用时才会初始化，而类初始化过程是非并行的，这些都由JLS能保证
  +  用enum实现单例
  +  还存在反射安全性问题：利用反射，可以访问私有方法，可通过加一个控制变量，该变量在getInstance函数中设置，如果不是从getInstance调用构造函数，则抛出异常；
+  Item 4: 将构造函数私有化，使得不能从类外创建实例，同时也能禁止类被继承  
  util类可能不希望被实例化，有其需求
+  Item 5: 避免创建不必要的对象
  +  提高性能：创建对象需要时间、空间，“重量级”对象尤甚；immutable的对象也应该避免重复创建，例如String；
  +  避免auto-boxing
  +  但是因此而故意不创建必要的对象是错误的，使用object pool通常也是没必要的
  +  lazy initialize也不是特别必要，除非使用场景很少且很重量级
  +  Map#keySet方法，每次调用返回的是同一个Set对象，如果修改了返回的set，其他使用的代码可能会产生bug
  +  需要defensive copying的时候，如果没有创建一个新对象，将导致很隐藏的Bug
+  Item 6: 不再使用的对象一定要解除引用，避免memory leak
  +  例如，用数组实现一个栈，pop的时候，如果仅仅是移动下标，没有把pop出栈的数组位置引用解除，将发生内存泄漏
  +  程序发生错误之后，应该尽快把错误抛出，而不是以错误的状态继续运行，否则可能导致更大的问题
  +  通过把变量（引用）置为null不是最好的实现方式，只有在极端情况下才需要这样；好的办法是通过作用域来使得变量的引用过期，所以尽量缩小变量的作用域是很好的实践；注意，在Dalvik虚拟机中，存在一个细微的bug，可能会导致内存泄漏，[详见](MemoryLeak.md)
  +  当一个类管理了一块内存，用于保存其他对象（数据）时，例如用数组实现的栈，底层通过一个数组来管理数据，但是数组的大小不等于有效数据的大小，GC器却并不知道这件事，所以这时候，需要对其管理的数据对象进行null解引用
  +  当一个类管理了一块内存，用于保存其他对象（数据）时，程序员应该保持高度警惕，避免出现内存泄漏，一旦数据无效之后，需要立即解除引用
  +  实现缓存的时候也很容易导致内存泄漏，放进缓存的对象一定要有换出机制，或者通过弱引用来进行引用
  +  listner和callback也有可能导致内存泄漏，最好使用弱引用来进行引用，使得其可以被GC
+  Item 7: 不要使用finalize方法
  +  finalize方法不同于C++的析构函数，不是用来释放资源的好地方
  +  finalize方法执行并不及时，其执行线程优先级很低，而当对象unreachable之后，需要执行finalize方法之后才能释放，所以会导致对象生存周期变长，甚至根本不会释放
  +  finalize方法的执行并不保证执行成功/完成
  +  使用finalize时，性能会严重下降
  +  finalize存在的意义
    +  充当“safety net”的角色，避免对象的使用者忘记调用显式termination方法，尽管finalize方法的执行时间没有保证，但是晚释放资源好过不释放资源；此处输出log警告有利于排查bug
    +  用于释放native peer，但是当native peer持有必须要释放的资源时，应该定义显式termination方法
  +  子类finalize方法并不会自动调用父类finalize方法（和构造函数不同），为了避免子类不手动调用父类的finalize方法导致父类的资源未被释放，当需要使用finalize时，使用finalizer guardian比较好：
    +  定义一个私有的匿名Object子类对象，重写其finalize方法，在其中进行父类要做的工作
    +  因为当父类对象被回收时，finalizer guardian也会被回收，它的finalize方法就一定会被触发

##Object的方法
尽管Object不是抽象类，但是其定义的非final方法设计的时候都是希望被重写的，finalize除外。
+  Item 8: 当重写equals方法时，遵循其语义
  +  能不重写equals时就不要重写
    +  当对象表达的不是值，而是可变的状态时
    +  对象不需要使用判等时
    +  父类已重写，且满足子类语义
  +  当需要判等，且继承实现无法满足语义时，需要重写（通常是“value class”，或immutable对象）
  +  当用作map的key时
  +  重写equals时需要遵循的语义
    +  Reflexive（自反性）: x.equals(x)必须返回true（x不为null）
    +  Symmetric（对称性）: x.equals(y) == y.equals(x)
    +  Transitive（传递性）: x.equals(y) && y.equals(z) ==> x.equals(z)
    +  Consistent（一致性）: 当对象未发生改变时，多次调用应该返回同一结果
    +  x.equals(null)必须返回false
  +  实现建议
    +  先用==检查是否引用同一对象，提高性能
    +  用instanceof再检查是否同一类型
    +  再强制转换为正确的类型
    +  再对各个域进行equals检查，遵循同样的规则
    +  确认其语义正确，编写测例
    +  重写equals时，同时也重写hashCode
    +  ！重写equals方法，传入的参数是Object
+  Item 9: 重写equals时也重写hashCode函数
  +  避免在基于hash的集合中使用时出错
  +  语义
    +  一致性
    +  当两个对象equals返回true时，hashCode方法的返回值也要相同
  +  hashCode的计算方式
    +  要求：equals的两个对象hashCode一样，但是不equals的对象hashCode不一样
    +  取一个素数，例如17，result = 17
    +  对每一个关心的field（在equals中参与判断的field），记为f，将其转换为一个int，记为c
      +  boolean: f ? 1 : 0
      +  byte/char/short/int: (int) f
      +  long: (int) (f ^ (f >> 32))
      +  float: Float.floatToIntBits(f)
      +  double: Double.doubleToLongBits(f)，再按照long处理
      +  Object: f == null ? 0 : f.hashCode()
      +  array: 先计算每个元素的hashCode，再按照int处理
    +  对每个field计算的c，result = 31 * result + c
    +  返回result
    +  编写测例
  +  计算hashCode时，不重要的field（未参与equals判断）不要参与计算
+  Item 10: 重写toString()方法
  +  增加可读性，简洁、可读、具有信息量
+  Item 11: 慎重重写clone方法
  +  Cloneable接口是一个mixin interface，用于表明一个对象可以被clone
  +  Contract
    +  x.clone() != x
    +  x.clone().getClass() ==  x.getClass()：要求太弱，当一个非final类重写clone方法的时候，创建的对象一定要通过super.clone()来获得，所有父类都遵循同样的原则，如此最终通过Object.clone()创建对象，能保证创建的是正确的类实例。而这一点很难保证。
    +  x.clone().equals(x)
    +  不调用构造函数：要求太强，一般都会在clone函数里面调用
  +  对于成员变量都是primitive type的类，直接调用super.clone()，然后cast为自己的类型即可（重写时允许返回被重写类返回类型的子类，便于使用方，不必每次cast）
  +  成员变量包含对象（包括primitive type数组），可以通过递归调用成员的clone方法并赋值来实现
  +  然而上述方式违背了final的使用协议，final成员不允许再次赋值，然而clone方法里面必须要对其赋值，则无法使用final保证不可变性了
  +  递归调用成员的clone方法也会存在性能问题，对HashTable递归调用深拷贝也可能导致StackOverFlow（可以通过遍历添加来避免）
  +  优雅的方式是通过super.clone()创建对象，然后为成员变量设置相同的值，而不是简单地递归调用成员的clone方法
  +  和构造函数一样，在clone的过程中，不能调用non final的方法，如果调用虚函数，那么该函数会优先执行，而此时被clone的对象状态还未完成clone/construct，会导致corruption。因此上一条中提及的“设置相同的值”所调用的方法，要是final或者private。
  +  重载类的clone方法可以省略异常表的定义，如果重写时把可见性改为public，则应该省略，便于使用；如果设计为应该被继承，则应该重写得和Object的一样，且不应该实现Cloneable接口；多线程问题也需要考虑；
  +  要实现clone方法的类，都应该实现Cloneable接口，同时把clone方法可见性设为public，返回类型为自己，应该调用super.clone()来创建对象，然后手动设置每个域的值
  +  clone方法太过复杂，如果不实现Cloneable接口，也可以通过别的方式实现copy功能，或者不提供copy功能，immutable提供copy功能是无意义的
  +  提供拷贝构造函数，或者拷贝工厂方法，而且此种方法更加推荐，但也有其不足
  +  设计用来被继承的类时，如果不实现一个正确高效的clone重写，那么其子类也将无法实现正确高效的clone功能
+  Item 12: 当对象自然有序时，实现Comparable接口
  +  实现Comparable接口可以利用其有序性特点，提高集合使用/搜索/排序的性能
  +  Contact
    +  sgn(x.compareTo(y)) == - sgn(y.compareTo(x))，当类型不对时，应该抛出ClassCastException，抛出异常的行为应该是一致的
    +  transitive: x.compareTo(y) > 0 && y.compareTo(z) > 0 ==> x.compareTo(z) > 0
    +  x.compareTo(y) == 0 ==> sgn(x.compareTo(z)) == sgn(y.compareTo(z))
    +  建议，但非必须：与equals保持一致，即 x.compareTo(y) == 0 ==> x.equals(y)，如果不一致，需要在文档中明确指出
  +  TreeSet, TreeMap等使用的就是有序保存，而HashSet, HashMap则是通过equals + hashCode保存
  +  当要为一个实现了Comparable接口的类增加成员变量时，不要通过继承来实现，而是使用组合，并提供原有对象的访问方法，以保持对Contract的遵循
  +  实现细节
    +  优先比较重要的域
    +  谨慎使用返回差值的方式，有可能会溢出

##Classes and Interfaces
+  Item 13: 最小化类、成员的可见性
  +  封装（隐藏）：公开的接口需要暴露，而接口的实现则需要隐藏，使得接口与实现解耦，降低模块耦合度，增加可测试性、稳定性、可维护性、可优化性、可修改性
  +  如果一个类只对一个类可见，则应该将其定义为私有的内部类，而没必要public的类都应该定义为package private
  +  为了便于测试，可以适当放松可见性，但也只应该改为package private，不能更高
  +  成员不能是非private的，尤其是可变的对象。一旦外部可访问，将失去对其内容的控制能力，而且会有多线程问题
  +  暴露的常量也不能是可变的对象，否则public static final也将失去其意义，final成员无法改变其指向，但其指向的对象却是可变的（immutable的对象除外），长度非0的数组同样也是有问题的，可以考虑每次访问时创建拷贝，或者使用`Collections.unmodifiableList(Arrays.asList(arr))`
  +  
  
+  Item 19: 仅仅用interface去定义一个类型，该类型有实现类，通过接口引用，去调用接口的方法
  +  避免用接口去定义常量，应该用noninstantiable utility class去定义常量
  +  相关常量的命名，通过公共前缀来实现分组