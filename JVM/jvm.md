# 深入拆解JVM
>深入拆解JVM课程详细内容见极客时间：https://time.geekbang.org/column/intro/108 
## 01 java代码怎么运行
- **java代码的运行方式**  
双击jar包运行、命令行、浏览器。依赖于JRE，java运行时环境。C++则不需要额外的运行时环境，编辑器把代码编译成机器码，直接可运行。
- **JRE中有什么**   
  包括运行java程序必需的组件：java虚拟机、java核心类库等
- **JDK中有什么**    
  包括JRE、以及开发和诊断工具
- **为什么java需要在jvm中运行**  
    java源代码编译成java字节码文件，然后运行在jvm中，而不是由编译器直接编译成机器码，直接运行在硬件上。  
    - 平台无关性    
    使用jvm java虚拟机屏蔽操作系统平台差异。在不同的操作系统平台提供不同的jvm，可以运行相同的字节码文件。
    - 提供托管环境  
    比如提供自动内存管理、垃圾回收等功能
- jvm具体怎样运行java字节码
    - jvm运行时数据区划分
        - 线程共享：
            - 堆：用于存放对象实例。
            - 方法区：用于方法被加载的类信息、常量、静态变量、JIT编译生成的代码等数据。
        - 线程私有：
            - java方法栈：每个方法开始执行时，都会在该线程对应的java方法栈中创建一个栈帧，栈帧中存放内容：局部变量表、操作数栈、动态链接、方法出口等。执行完该方法，则对应栈帧从栈中移出。栈帧的大小是提前计算好的，且不要求在内存空间中连续分布。
            - 本地方法栈：用于C++编写的native方法，类似java方法栈。
            - PC寄存器：存放当前线程执行字节码的行号。
    - 虚拟机角度  
        1. 把编译好的java字节码文件加载到jvm中
        2. 把加载好的java类存放到jvm的方法区中
        3. jvm执行方法区中的代码
    
    - 硬件角度  
        java字节码无法直接运行，需要jvm把java字节码翻译成机器码。在HotSpot中，翻译方式有两种： 
        - 解释执行：逐条将字节码翻译成机器码，然后执行。
        - 及时编译（JIT）：将一个方法中包含的所有字节码编译成机器码之后再执行。
          
        优缺点对比：  
        
        | | 优点 | 缺点 |
        |:---:|:---:|:---:|
        |解释执行|不需要等待编译<br>能够让程序快速启动执行|执行效率较低|
        |及时编译|执行效率高|编译需要耗费相对较长时间|
        
        HotSpot默认采用混合模式：先解释执行字节码，然后将反复执行的热点代码，以方法为单位进行及时编译。
- jvm运行效率  
HotSpot采用多种技术提升启动性能、以及峰值性能。技术就包括及时编译。  
及时编译建立在二八定律假设上，百分之二十的代码占用了百分之八十的计算资源。对于百分之八十不常用的代码，使用解释执行方法运行；对于百分之二十的热点代码，采用及时编译成机器码，提高运行效率。  
及时编译相比于静态编译具有的优势：  
及时编译能获取到程序运行时的统计信息，根据运行时信息可以做出更多的优化，提高程序运行效率。  
例子：  
java面向对象具有多态的特性，对于一个虚方法调用，可能会对应多个目标方法，其实往往程序运行中只使用到其中一个，那么可以在及时编译时，直接使用目标方法，省去虚方法调用开销。  

HotSopt内置多个及时编译器：C1、C2、Graal（java10正式引入的实验性及时编译器）。  
 - C1：Client编译器，面向对启动性能有要求的客户端GUI程序。优化手段相对简单，编译时间较短，执行效率相对较低。
 - C2: Server编译器，面向对峰值性能有要求的服务端程序。优化手段相对复杂，编译时间较长，但执行效率较高。  
 java7开始，HotSpot默认采用分层编译，热点方法首先被C1编译，之后热点中的热点方法会被C2编译。  
 HotSpot即使编译会在单独的编译线程中进行，避免对应用的正常运行造成影响。
 HotSpot会根据CPU数量设置编译线程的数量，然后编译线程的数量按照1：2比例分配给C1和C2及时编译器。
          
## 02 Java的基本类型
java基本类型：byte、short、char、boolean、int、long、float、double。
  
类型|值域|默认值|虚拟机内部符号    
:---:|:---:|:---:|:---:
boolean|{false,true}|false|Z
byte|[-128,127] (-2<sup>7</sup>,2<sup>7</sup>-1)|0|B
short|[-32768,32767] (-2<sup>15</sup>,2<sup>15</sup>-1)|0|S
char|[0,65535] (0,2<sup>16</sup>)|'\u0000'|C
int|[-2<sup>31</sup>,2<sup>31</sup>-1]|0|I
long|[-2<sup>63</sup>,2<sup>63</sup>-1]|0L|L
float| |+0.0F|F
double| | +0.00D|D
默认值在内存中都是0  
- boolean类型  
java语言规范中boolean类型只有两种值：true、false。boolean类型被映射成init类型：true映射成1，false映射成0。  
java虚拟机规范要求java编译器也要遵守这个编码规范，用整数相关字节码实现逻辑运算、boolean类型条件跳转。  
可以通过使用字节码编辑工具修改boolean类型变量的值为0、1之外的整数，这样字节码工具比如：AsmTools(OpenJDK中包含)、ASM等。  
- java基本类型的大小  
    - java栈  
    jvm每调用一个java方法，创建一个栈帧。栈帧两个主要组成部分：局部变量区、字节码操作数栈。局部变量区：局部变量、实例方法this指针、  
    方法参数。jvm规范中，局部变量区相当于一个数组，long、double占用两个数组单元存储、其他基本数据类型和引用类型值占用一个单元，一个单元   
    占用（32位HotSpot中占用4个字节、64位占用8个字节）
    - java堆  
    在java堆中byte、char、short三种类型字段和数组单元分别占用一个字节、两个字节、两个字节。boolean字段占用一个字节，boolean数组使用byte  
    数组实现。为了保证java堆中的boolean类型字段值合法，HotSpot在存储时显式进行掩码操作，只取最有一位值存放boolean字段或数组中（奇数则为1，  偶数为0）。
      
jvm的算数运算几乎全部依赖于操作数栈，需要将jvm堆中的boolean、byte、short加载到操作数栈中，然后把操作数栈中的值当做int类型运算。  
## 03 jvm如何加载java类  
jvm把java类的字节码变成内存中的类，大致经过了三个步骤：加载、连接、初始化。  
需要jvm从字节码变成内存中的类类型（必须经过加载过程的类型）包括：类、接口。java语言的类型可分为两大类：基本类型、引用类型。java基本类型是由jvm预先定义好的，数组类由jvm直接生成。泛型参数会在编译过程中被擦除。  
java字节码字节流来源：java编译器编译java源码文件生成的class文件、程序内部直接生成、从网络中获取的字节流。这些字节流会由jvm加载到jvm中，然后成为内存中的类或接口。  
### 加载
加载是指查找字节流。查找字节流需要使用类加载器（Class Loader）来完成。  
- 启动类加载器：Bootstrap ClassLoader 由C++代码实现。负责加载JAVA_HOME\lib目录下、或者使用-Xbootclasspath指定的路径中，包含的启动类加载器可以识别的类（按照文件名称识别），比如rt.jar。
- java.lang.ClassLoader子类
    - 扩展类加载器：Extension ClassLoader 由sun.misc.Launcher$ExtClassLoader实现，负责加载JAVA_HOME\lib\ext目录下、或者java.ext.dirs系统变量指定的路径中的所有类库。
    - 应用类加载器: System ClassLoader 由sun.misc.Launcher$AppClassLoader实现，负责加载classpath指定的应用程序类库。
主要的三个类加载器层次关系：应用类加载器 -》 扩展类加载器 -》 启动类加载器  
类加载器之间的层次关系称为双亲委派模型，要求：除了启动类加载器之外的所有类加载器都要有自己的父类加载器。具体双亲委派加载类的过程：
        - 将加载请求发送给父类加载器去完成类加载过程
        - 父类加载器同样会将加载请求交给自己的父类加载器
        - 如果父类加载器没有加载到请求加载的类，则尝试自己去加载  
java9中，扩展类加载器改成了平台类加载器，引入了模块系统的概念。类加载器起到命名空间的作用，在jvm中，一个类的唯一标示为：加载它的类加载器 + 类的全名。  
### 链接
链接是把创建的类合并到jvm中，让类可以执行的过程。链接过程分三步走：验证、准备、初始化。  
- 验证：校验被加载的类是否满足jvm的约束要求。比如验证字节码开头四个字节应该是魔数（CAFE BABE）。  
> 魔数通常用于标示文件格式类型。具体魔数的介绍：https://en.wikipedia.org/wiki/Magic_number_%28programming%29   
  java字节码使用CAFE BABE作为魔数的原因：https://en.wikipedia.org/wiki/Java_class_file 的Magic Number部分  
    
- 准备：为被加载器类的静态字段分配内存。
- 解析：将符号引用解析成实际引用。   
符号引用：在类的class文件被加载到jvm中之前，类还不知道自己使用的其他类、其他类的方法、字段在内存中的地址。java编译器在编译java源代码时，会将这些对类、方法、字段的引用使用一个符号引用代替。比如：   
对于一个方法调用，java编译器会生成一个符号引用来代替，这个符号引用包含：目标方法所在的类名、目标方法名称、参数类型、返回值类型。  
解析的过程就是把这种符号引用解析成实际引用，如果涉及到引用的类或者方法和字段所属的类还未被加载，则触发对这种未被加载的类的加载。  
jvm规范规定在执行字节码之前，要求完成涉及到的符号引用的解析，没有要求必须要在链接阶段完成解析。  
### 初始化
java代码中，初始化一个静态字段，可以直接赋值，也可以在静态代码段中进行赋值。
- 直接赋值的final修饰的基本类型或者字符串类型静态字段，java编译器会标记该字段为ConstantValue，该字段的初始化直接由jvm完成。  
- 除了上面这种情况之外的字段直接赋值、所有静态代码块中代码，会被java编译器放到同一个方法中，这个方法名称是<clinit>。这个方法jvm通过加锁来保证仅被执行一次。
jvm规范中规定的触发初始化的多种条件：  
- jvm启动时，初始化用户指定的主类  
- 通过new关键字创建类的实例对象，初始化这个类  
- 静态方法调用，静态方法所在的类会被初始化
- 静态字段访问，静态字段所在的类会被初始化
- 子类初始化，子类的父类需要被初始化  
- 接口定义了default方法，直接实现或者间接实现该接口的类初始化时，会触发该接口的初始化   
- 使用反射api调用类时，触发这个类初始化  
单例模式的例子：
```
public class Singleton {
    private Singleton() {}
    private static class LazyHolder 
        static final Singleton INSTANCE = new Singleton();
    }
    public static Singleton getInstance(){
        return LazyHolder.INSTANCE;
    }
}
```
类的初始化是线程安全的，并且只会执行一次，所以可以利用类初始化的特性实现延迟加载单例模式。  
## 04 05 jvm如何执行方法调用？  
### 重载和重写  
- 重载  
    - 同一个类中：多个方法，名字相同，参数类型不同，这些方法直接的关系称为重载。  
    - 子类和父类中：子类中定义了和父类中非私有方法同名的方法，且两个方法的参数类型不同，则这两个方法之间关系为重载。  
    
    方法重载在编辑过程中就可以完成识别。具体识别具体的目标方法的规则：  
    - 在不考虑自动拆箱装箱和可变长参数（比如：Object ... xxx）情况下，匹配目标方法。  
    - 在考虑自动拆箱装箱、不考虑可变长参数的情况下，匹配目标方法。  
    - 在考虑自动拆箱装箱、考虑可变长参数的情况下，匹配目标方法。  
    - 优先考虑更加具体的类型，比如重载的方法中，一个方法的参数类型为String，另一个的参数类型是Object，优先考虑String类型的方法。  
- 重写
子类和父类中方法的关系，如果子类中存在一个方法和父类中的一个非私有非静态的方法同名，且这两个方法参数类型相同，则这两个方法之间的关系为重写，子类中的方法重写了父类中的方法。  
如果子类的方法和父类的静态非私有方法同名，且参数类型相同，则子类的方法**隐藏**了父类的方法。  
### jvm的静态绑定和动态绑定  
jvm中的静态绑定：在解析时能够解析出来目标方法称为静态绑定。  
jvm中的动态绑定：需要在运行过程中根据调用者的动态类型识别目标方法称为动态绑定。  
java字节码中和调用相关的五种指令：  
- invokestatic：调用静态方法
- invokespecial：调用实例私有方法、构造器；super关键字调用父类实例方法、构造器；所实现接口的默认方法（default）  
- invokevirtual：调用非私有实例方法  
- invokeinterface：调用接口方法  
- invokedynamic：调用动态方法  
静态绑定：invokestatic、invokespecial、final修饰的非私有实例方法 invokevirtual  
动态绑定：invokeinterface、invokevirtual  
### 符号引用  
java代码编译过程中，不知道目标方法的具体内存地址，编译器会使用符号来表示目标方法。符号引用包括：目标方法所在的类或接口名字、方法名、方法描述符。   
符号引用存放在class文件的常量池中，使用 javap -v xxx 命令可以查看到每个类的常量池。   
接口符号引用：目标方法是接口方法  
非接口符号引用：目标方法非接口方法   
非接口符号引用，假设符号引用对应的类是C，则jvm解析符号引用过程：  
1. 在C中查找符合名字及方法描述符的方法。  
2. 1中没有找到，则在C的父类中继续找，一直找到Object类。  
3. 2中没找到，则在C直接或间接实现的接口中找，要求找到的目标方法必须是非私有、非静态的。如果目标方法在间接实现的接口中，则需要满足C和该接口直接没有其他符合条件的目标方法。如果存在多个符合条件的  
目标方法，则任意返回其中的一个。  

接口符号引用，假设符号引用对应接口I，则jvm解析符号引用过程：
1. 在I中查找符合名字和方法描述符的方法。  
2. 1没有找到，则在Object类中公有实例方法中找。  
3. 2没找到，则在I的超接口中找，具体要求和非接口符号引用解析一样。  
经过解析，符号引用被解析成实际引用。静态绑定的符号引用，可以解析得到一个指向目标方法的指针。动态绑定的符号引用，则解析得到一个方法表的索引。   

### 虚方法调用  
invokevirtual和invokeinterface两种指令都属于jvm虚方法调用。  
jvm采用了空间换时间的方法实现方法调用的动态绑定：方法表。  
### 方法表  
invokevirtual对应虚方法表 virtual method table，invokeinterface对应接口方法表 interface method table。  
方法表本质上是一个数组，每个数组元素指向一个当前类或祖先类中非私有实例方法。这些方法可以是具体可执行的方法，也可能是没有相应字节码的抽象方法。  
方法表的满足的规则：   
1. 子类方法表中包含了所有父类方法表中的所有方法。  
2. 子类方法在方法表中的索引值和这个方法所重写的父类方法索引值相同。  
需要动态绑定的符号引用，在解析成实际引用的时候，解析成方法表中索引值（不仅仅）。  
动态绑定方法执行过程：jvm获取调用者的实际类型，然后在实际类型中获取对应类的方法表，然后根据之前解析得到的索引值，获取方法表元素指向的实例方法，即目标方法。  

仅仅在及时编译、及时编译最坏情况的时候会使用方法表动态绑定。  
jvm进一步优化动态绑定方法执行，提供了两种方法：内联缓存、方法内联。  
### 内联缓存  
内联缓存是一种加快动态绑定的优化技术。具体是将类型的动态类型、调用的方法 所对应的 目标方法缓存起来。 缓存之后，后续执行：  
 1. 找当前实际类型、调用方法 是否被内联缓存，如果找到，则直接获取缓存的目标方法执行。  
 2. 如果1中没有找到缓存，则执行方法表动态绑定过程。  
在针对多态的优化技术中，涉及的三个术语：  
  1. 单态：仅有一种状态的情况。  
  2. 多态：数量有限数量种类的情况。  
  3. 超多态：更多种状态的情况。指定一个种类数值，超过这个数值，则成为超多态。  
相应的，内联缓存可以分为：单态内联缓存、多态内联缓存、超多态内联缓存。  
  1. 单态内联缓存：只缓存一种动态类型以及它所对应的目标方法。查找内联缓存时，只需要判断当前缓存的一份类型是不是当前执行方法实例的类型。    
  2. 多态内联缓存：缓存多个动态类型及其目标方法。超找内联缓存时，需要遍历缓存的所有动态类型，判断当前是否是当前执行方法实例的类型。 这种情况，会把一个类型的热点动态类型放在靠前的位置。  
jvm中为了节省内存空间，只采用单态内联缓存。而且在实践中，大部分虚方法调用都是单态的，只有一种动态类型。  
在多态情况下，采用单态内联缓存，如果查找动态类型缓存失败的情况，进行方法表动态绑定，可以做两种选择：  
1. 替换现有的内联缓存，修改成当前动态类型及其目标方法。这样做的能提高执行效率的前提是满足局部性要求，要求替换缓存之后的一段时间内，调用该方法的实例类型都是当前的实例类型。比如有两种动态类型A,B，
  调用方法轮流执行，这样会造成优化失效，并且还带来了写缓存的时间和空间开销。  
2. 放弃优化，直接走动态方法表绑定。  
对于很简单的方法，方法调用的开销可能比方法本身执行开销还要大，这种情况在及时编译的时候，可以通过方法内联消除方法调用的固定开销。  
## 06 jvm如何处理异常？  
异常处理两个重要组成：抛出异常、捕获异常。两个重要组成共同控制了程序控制流的非正常转移。  
抛出异常可分为两种形式：  
- 显示抛出异常  
程序使用throw关键字手动抛出异常，抛出异常的主体是程序。    
- 隐式抛出异常
jvm执行过程中，遇到无法继续执行的异常状态，自动抛出异常。比如数组访问下标越界，jvm抛出数组索引越界异常。抛出异常的主体是jvm。  
捕获异常涉及三种代码块：  
- try代码块：标记需要进行异常监控的代码。 
- catch代码块：紧跟在try代码块之后，用来捕获try代码块中触发的某种执行类型的异常。 声明了捕获异常的类型，同时提供了针对该类异常的处理逻辑。java中一个try代码块儿可以跟多个catch块儿，  
来捕获不同类型的异常。jvm会从上往下匹配异常处理逻辑，要求上面的异常类型不能覆盖包含下面的异常类型，编译器会校验这个限制。  
- finally代码块：跟在catch代码块儿之后，或者直接跟在try代码块儿之后，用于声明一段必须执行的代码段，为了避免因为异常，跳过了必须执行的代码，比如关闭已经打开的系统资源。  
### 异常的基本概念  
java中所有的异常都是Throwable类或其子类的实例。Throwable类分为两大类：  
- Error
程序不应该捕获的异常，当程序出现了Error，代表的是程序的执行状态已经没办法恢复，只能终止线程，甚至是终止整个jvm。  
- Exception  
程序可能需要捕获且处理的异常。  
RuntimeException：Exception的子类，表示程序已经无法继续执行，但是可以通过捕获处理使程序继续执行，数组越界异常就是一种RuntimeException。  
RuntimeException是非检查异常（unchecked exception），其他的异常为检查异常（checked exception）。  
检查异常需要程序显式地捕获异常，或者在方法声明中增加throws关键字。  
异常实例对象的构造代价很高：  
发生异常时，jvm会生成异常的栈轨迹（stack trace）。   
生成栈轨迹：遍历当前线程的java栈帧，记录下各种调试信息（栈帧指向的方法名称、方法所在类名、文件名、以及触发异常的代码行数）。
### jvm如何捕获异常  
在编译生成的字节码中，每个方法对应一个异常表。  
异常表中的每一条数据，包含了：from指针，to指针，target指针以及所捕获的异常类型。异常表中的每条数据对应了一个异常处理器。  
from指针、to指针、target指针都是字节码索引，指向字节码，用于定位字节码。from和to限定了异常处理器监控的范围，target指针指向了异常处理器的开始位置。  
程序发生异常时，jvm会遍历当前方法的异常表，从上往下开始：  
1. 如果触发异常的字节码的索引值在某个异常处理器的监控范围之内，jvm会继续判断异常类型是否也匹配，匹配则将从target指向的字节码开始执行。  
2. 如果1中范围或异常类型无法匹配，则继续在异常表中遍历。  
3. 如果在当前方法的异常表中无法匹配，则该方法的java栈帧会弹出，然后再调用当前方法的方法中，继续重复遍历匹配的过程。  
finally代码块在编译时做了特殊处理，java编译器目前的做法是：把finally代码块的内容分别放到对应的try-catch代码块所有正常执行路径出口处和所有异常处理路径的出口处。  
```
try {
 try{
    // 1
    throw new RuntimeException("exception");
 } catch (Exception e) {
    // 2 
    System.out.println("catch 1");
    // 2'
    throw e;
 } finally {
    // 3
    System.out.println("finally 1");
    // 3'
    return;
 }    
} catch (Exception e) {
   // 4 
   System.out.println("catch 2");
   ....   
} finally {
    // 5
    System.out.println("finally 2");
}
```  
上面的代码示例，执行顺序：1->2->3->3'->5，返回结果：  
```
catch 1
fianlly 1
finally 2
```  
对于异常路径，java编译器会生成一个或多个异常表条目，监控整个try-catch代码块，并且捕获所有种类的异常类型，这些异常表条目的target指向复制的finally代码块。  
在finally代码块最后，java编译器会重新抛出所捕获的异常。但是在finally块儿中return了，则异常不会抛出了。  

## 07 JVM如何实现反射  
反射是java语言中一个相当重要的特性，可以实现程序在运行过程中获取类的所有方法、字段等信息、支持调用类的方法、获取字段的值。  
例子：  
- 可以通过Class对象获取该类的所有方法，可以通过Method.setAccessible方法绕过java语言的访问权限，实现在私有方法类之外的类中被调用。  
- IDE自动提示调用对象的方法和字段  
- java调试器，能够在调试过程中枚举一个对象的所有字段值。  
- java可配置的通用框架，为了保证扩展性，一般会使用java反射机制根据配置文件加载配置的类。比如Spring IOC容器。  
### 反射调用的实现  
java.lang.reflect.Method#invoke源码：  
```
public Object invoke(Object obj, Object... args)
        throws IllegalAccessException, IllegalArgumentException,
           InvocationTargetException
    {
        ...// 权限检查
        MethodAccessor ma = methodAccessor;             // read volatile
        if (ma == null) {
            ma = acquireMethodAccessor();
        }
        return ma.invoke(obj, args);
    }
```
方法的发射调用具体实现委托给MethodAccessor来处理。MethodAccessor为接口，具体源码：  
```
public interface MethodAccessor {
    Object invoke(Object var1, Object[] var2) throws IllegalArgumentException, InvocationTargetException;
}
```
MethodAccessor有两个具体的实现：通过本地方法来实现反射调用（NativeMethodAccessorImpl）、委派模式实现（DelegatingMethodAccessorImpl）  
每个Method实例第一次反射调用时，会生成一个委派实现，委派实现委派的是本地方法调用实现。      
本地方法实现：进入jvm内部，可以获取Method对象所指向的目标方法地址，直接调用目标方法即可。  
Method.invoke->DelegatingMethodAccessorImpl->NativeMethodAccessorImpl native method->目标方法  
为什么不直接是 Method.invoke->NativeMethodAccessorImpl native method->目标方法呢？  
因为java反射调用机制还提供了一种动态生成字节码的实现，直接使用 invoke指令调用目标方法。为了能在本地实现和动态字节码生成实现两个实现之间切换，所以采用了  
委派实现。  
两者对比：
 
|  |优点|缺点|
:---:|:---:|:---:  
本地实现|不需要动态生成字节码的等待耗时 |运行效率较低 |  
动态实现| 运行效率高，大概要比本地实现快20倍|动态生成字节码耗时较长 |
  
本地实现效率低的原因：因为要调用本地方法，需要从java切换到c++，然后再返回到java。  
动态实现效率高的原因：直接生成字节码，不需要来回切换的过程。  
可以设置一个阈值，未超过阈值之前，采用本地实现，超过阈值之后可采用动态实现，动态生成字节码，并将委派实现的对象切换到动态实现。   
阈值设置参数：-Dsun.reflect.inflationThread。  
java反射调用可以通过设置，直接使用动态实现，不使用委派实现和本地实现。设置参数为：-Dsun.reflect.noInflation = true。  
NativeMethodAccessorImpl 源码：  
```
public Object invoke(Object obj, Object[] args)
              throws IllegalArgumentException, InvocationTargetException
   {
        if (++numInvocations > ReflectionFactory.inflationThreshold()) {
            // 调用次数超过了设置的阈值，则会创建动态实现
            MethodAccessorImpl acc = (MethodAccessorImpl)
               new MethodAccessorGenerator().
                  generateMethod(method.getDeclaringClass(),
                                     method.getName(),
                                     method.getParameterTypes(),
                                     method.getReturnType(),
                                     method.getExceptionTypes(),
                                     method.getModifiers());
            // 修改委派实现为动态实现                         
            parent.setDelegate(acc);
        }
        return invoke0(method, obj, args);
    }
```
ReflectionFactory创建MethodAccessor对象的源码：  
```
public MethodAccessor newMethodAccessor(Method var1) {
        checkInitted();
        if(noInflation) {
            // 默认是false，修改成true，则直接使用动态实现
            return (new MethodAccessorGenerator()).generateMethod(var1.getDeclaringClass(), var1.getName(), var1.getParameterTypes(), var1.getReturnType(), var1.getExceptionTypes(), var1.getModifiers());
        } else {
            // 默认采用委派实现，委派的是本地方法实现
            NativeMethodAccessorImpl var2 = new NativeMethodAccessorImpl(var1);
            DelegatingMethodAccessorImpl var3 = new DelegatingMethodAccessorImpl(var2);
            var2.setParent(var3);
            return var3;
        }
    }
```
Method.invoke方法中首次反射调用创建方法访问对象的方法 `acquireMethodAccessor()`代码： 
```
private MethodAccessor acquireMethodAccessor() {
        // First check to see if one has been created yet, and take it
        // if so
        MethodAccessor tmp = null;
        if (root != null) tmp = root.getMethodAccessor();
        if (tmp != null) {
            methodAccessor = tmp;
        } else {
            // Otherwise fabricate one and propagate it up to the root
            tmp = reflectionFactory.newMethodAccessor(this);
            setMethodAccessor(tmp);
        }
        return tmp;
    }
```
### 反射调用开销  
Class.forName方法会调用本地方法，相对比较耗时。  
Class.getMethod方法会遍历该类的公有方法，如果没有匹配到，会遍历父类的公有方法。  
在实践中，一般会通过程序缓存Class.forName、Class.getMethod结果，用空间换时间。  
Method.invoke方法是一个变成参数方法，在字节码层面使用的参数为Object数组，如果传参是一个，所以会构造一个Object数组，会有一定的性能消耗。  
Object数组不能存储基本类型，那么如果参数是一个int，那么会涉及到装箱操作，且默认jvm缓存了[-128,127]范围中的整数对应的Integer对象，如果在这个范围之外，则需要新建一个Integer对象。  
虚方法调用的动态实现，Method.invoke方法对应一个GeneratedMethodAccessor。在实际生产环境中，往往会有多个不同的反射调用，对应多个GeneratedMethodAccessor。   
在设置关闭inflation或者超过设定的调用阈值之后生成动态调用时，ReflectionFactory的newMethodAccessor方法中、本地方法NativeMethodAccessorImpl的invoke
方法中，都会调用类似的方法：`(new MethodAccessorGenerator()).generateMethod(this.method.getDeclaringClass(), this.method.getName(), this.method.getParameterTypes(), this.method.getReturnType(), this.method.getExceptionTypes(), this.method.getModifiers());`，具体MethodAccessorGenerator的generateMethod方法源码如下：  
```
public MethodAccessor generateMethod(Class declaringClass,
                                            String name,
                                               Class[] parameterTypes,
                                               Class   returnType,
                                               Class[] checkedExceptions,
                                               int modifiers)
  {
         return (MethodAccessor) generate(declaringClass,
                                           name,
                                           parameterTypes,
                                           returnType,
                                           checkedExceptions,
                                           modifiers,
                                           false,
                                           false,
                                           null);
  }
   
private MagicAccessorImpl generate(final Class declaringClass,
                                            String name,
                                            Class[] parameterTypes,
                                            Class   returnType,
                                            Class[] checkedExceptions,
                                            int modifiers,
                                            boolean isConstructor,
                                            boolean forSerialization,
                                            Class serializationTargetClass){
       ... // 字节码生成                                     
  }
```
因为jvm会对invokevirtual或invokeinterface记录调用者的具体类型，称为profile，但是无法记录太多的类型，如果反射调用涉及的类型太多，会造成反射调用没有被内联。  
可以通过jvm参数 -XX:TypeProfileWidth参数设置每个调用能够记录的类型数量，默认值是2。   
## 08 09 jvm如何实现invokedynamic  
java 7引入了一条新指令 invokedynamic。这个指令的调用机制抽象出调用点这一概念，并允许应用程序将调用点链接到任意符合条件的方法上。  
java 7引入了更加底层、更加灵活的方法抽象：方法句柄MethodHandle。  
### 方法句柄  
方法句柄是一个强类型的，能够被直接执行的引用。该引用可以指向常规的静态方法或实例方法、构造器、字段。   
当方法句柄指向字段时，方法句柄实际指向了包含字段访问字节码的虚构方法，相当于目标字段的getter和setter方法，但是并不是已有的getter和setter方法，毕竟无法保证程序提供的getter和setter方法实际访问的是不是目标字段，不可靠。  
方法句柄的类型（MethodType）：由参数类型、返回类型组成。方法句柄的类型用来确认方法句柄是否适配方法的，并不关心方法句柄所指向方法的方法名、以及所在类的类名。  
方法句柄创建：通过MethodHandles.Lookup类完成。  
- 使用Method查找
- 根据类、方法名、方法句柄类型查找  
需要区分具体调用类型：
invokestatic调用的静态方法，需要使用Lookup.findStatic方法；  
invokevirtual调用的实例方法，以及invokeinterface调用的接口方法，需要使用Lookup.findVirtual方法；  
invokespecial调用的实例方法，需要使用findSpecial方法。  
调用方法句柄和原本方法的调用指令是一致的。对于原本使用invokevirtual调用的方法句柄，也会采用动态绑定；对于原本使用invokespecial调用的方法句柄，也会采用静态绑定。  
方法句柄调用有权限检查，检查是在方法句柄创建时完成，在实际调用过程中，不会检查方法句柄的权限。对比反射API的每次调用检查权限，可以节省重复校验权限的开销。  
### 方法句柄操作  
- invokeExact，需要严格匹配参数类型。如果方法句柄调用发现参数类型不匹配，会抛出方法类型不匹配异常。  
签名多态 signature polymorphism，方法句柄API有一个特殊注解类@PolymorphismSignature。  
java编译器会在遇到被签名多态注解类注解的方法调用时，根据调用传入的参数类型来生成方法描述符，而不再是目标方法的参数类型。
- invoke调用方式，是一个签名多态性的方法，可以自动适配参数类型。  
invoke会调用Method.asType方法，生成一个适配方法句柄，对传入参数进行适配，再调用原方法句柄；调用原方法句柄的返回值，也会再适配返回给调用者。  
方法句柄支持增删改参数的操作：通过生成另一个方法句柄实现。
- 增：在传入参数中，新增额外的参数，然后调用另一个方法句柄。对应API MethodHandle.bindTo方法。  
- 删：将传入的部分参数抛弃掉，再调用另一个方法句柄。对应API MethodHandles.dropArguments方法。  
- 改：MethodHandle.asType  
### 方法句柄的实现  
-XX:+ShowHiddenFrames参数设置打印被jvm隐藏的栈信息。  
方法句柄invokeExact调用过程：
jvm对invokeExact调用做特殊处理，调用一个共享的、和方法句柄类型相关的特殊适配器（LambdaForm）中。  
 - 调用Invokers.checkExactType方法检查参数类型  
 - 调用Invokers.checkCustomized方法，在方法句柄调用执行次数超过一定阈值（默认127）之后进行优化。  
 - 调用方法句柄的invokeBasic方法。jvm也会对invokeBasic调用做特殊处理，会调用方法句柄本身所持有的适配器（是一个LambdaForm）。
   - 获取方法句柄中的MemberName类型字段，然后调用linkToStatic方法，参数中有MemberName类型字段。jvm会对linkToStatic调用做特殊处理，根据MemberName类型字段中存储的方法地址或  
   方法表的索引，直接调用目标方法。  
### invokedynamic指令  
invokedynamic是java 7引入的一条新指令，用来支持动态语言的方法调用。在运行过程中，每一条invokedynamic指令绑定一个调用点，会调用调用点所链接的方法句柄。  
在第一次执行invokedynamic指令时，jvm会调用指令对应的启动方法（Bootstrap Method），来生成调用点，并把调用点绑定到该指令上。  
之后的运行过程中，jvm会直接调用绑定的调用点链接的方法句柄。  
启动方法也是通过方法句柄来指定的，这个方法句柄 返回值：调用点（CallSite） 固定参数：Lookup类实例、表示目标方法名字的字符串、调用点能够链接的方法句柄类型。  
除了三个固定的参数之外，还可以有其他的参数，用来辅助生成调用点，或者定位链接的目标方法。  
后续详细分析。  
## 10 java对象的内存布局  
java新建对象的方式：new语句、反射、Object.clone方法、反序列化、Unsafe.allocateInstance方法。  
Object.clone方法、反序列化通过直接复制已有数据，来初始化新建对象的实例字段。  
Unsafe.allocateInstance没有初始化实例字段。  
new语句和反射机制，通过调用构造器初始化实例字段。new语句，编译生成的字节码中包含：请求内存的new指令、调用构造器的invokespecial指令。   
java构造器约束：  
1. 一个类如果没有定义任何构造器，java编译器会自动添加一个无参构造函数。  
2. 子类构造器需要调用父类构造器，如果父类存在无参构造函数，则java代码里子类对父类构造器调用是隐式的，在java编译器编译时会自动添加对父类构造器的调用；如果父类中没有无参构造器，则需要  
  子类显式调用父类构造器调用，并且要求调用父类构造器的代码作为子类构造器中国第一条语句，这么要求是为了优先初始化从父类中继承的父类字段（可以通过字节码注入等方法绕开限制）。  
通过new指令新建出来的对象，包含了所有父类中的实例字段，即使是子类无法访问的父类私有字段、子类实例字段隐藏了父类的同名实例字段，子类实例也会为这些字段分配内存。  
### 压缩指针  
在jvm中，每个对象都有一个对象头，对象头由标记字段、类型指针构成。  
- 标记字段：jvm关于该对象的运行时数据，比如：哈希码、GC信息、锁信息     
- 类型指针：指向该对象的类  
在64位的jvm中，对象头标记字段占64位，类型指针占64位，对象头总共占128位 16个字节。  
为了尽量减少对象头的内存占用，64位虚拟机引入压缩指针的概念，jvm虚拟机选项：-XX:+UseCompressedOops，jvm默认开启指针压缩。  
指针压缩将原来的64位java对象指针压缩到32位，压缩指针可以作用于：对象头的类型指针、引用类型字段、引用类型数组。那么对象头的占用字节数从 16 变成 12。  
压缩指针的原理：  
一个寻址单位包含多个基本单元，缩小寻址单位的范围，从而减小用于记录地址的字节数。  
默认jvm堆中对象的其实地址都需要对齐到8的倍数，那么如果一个对象占用不到8N个字节，那么剩余的空间就浪费了，浪费的空间称为对象之间的填充 padding。牺牲部分空间，换取每个对象存储对象头的占用空间。  
jvm虚拟机中的32位压缩指针可以寻址字节数：2<sup>32</sup> * 8 = 2<sup>35</sup> = 32GB，超过32GB的地址空间会关闭压缩指针。  
32位压缩指针在解析地址时，会将指针左移三位，然后加上固定的偏移量。  
可以通过调整内存对齐的参数-XX:ObjectAlignmentInBytes，增大这个值，可以进一步提升寻址范围，但是相应的可能增加了对象之间的padding，浪费的空间 > 压缩指针节省的空间。  
内存对齐跟是否开启压缩指针没有强关系，即使关闭了压缩指针，jvm还是需要内存对齐的，比如jvm要求long字段、double字段、压缩指针关闭情况下非压缩指针引用字段地址为8的倍数。  
内存对齐可以出现在对象之间，也会出现在一个对象的字段之间。  
字段内存对齐的一个原因：保证一个字段出现在只出现在一个CPU缓存行中，避免一个字段落到两个缓存行中，避免这个字段的修改污染了两个CPU缓存行，避免获取该字段需要读取两个缓存行。  
一个字段需要两个缓存行存储的情况，会影响程序的执行效率。  
### 字段重排列  
字段重排列：jvm重新调整确定字段的先后顺序，来实现内存对齐的目的。jvm有三种排列方法，-XX:FieldsAllocationStyle，默认是1。三种排列方法都会遵循的规则：  
1. 如果一个字段占C个字节，那么该字段的偏移量需要对齐到NC，偏移量为该字段的地址和对象起始地址的差值。  
2. 子类继承的字段的偏移量，需要和父类中对应的字段偏移量保持一致。  
在具体实现中，jvm会对齐类字段的起始位置，对于使用压缩指针的64位虚拟机，子类第一个字段需要对齐到4N；对于关闭了压缩指针的64位虚拟机，子类的第一个字段需要对齐到8N。  
伪共享（false sharing）  
本来逻辑上不需要共享的字段，但是因为在同一个CPU缓存行中，导致构成实际的共享。  
假设两个线程分别访问同一个对象中的不同的volatile字段，逻辑上并没有共享，但是如果两个字段在同一个缓存行中，那么其中一个字段的写操作都会导致缓存行的写回，造成了实际上的共享。  
java 8 引入了一个新的注解@Contented，用来解决伪共享问题。jvm会让不同的@Contended注解的字段在独立的缓存行中。需要使用jvm参数-XX:-RestrictContended。    
## 11 12 垃圾回收
jvm的自动内存管理，把原本需要由开发人员手动回收的内存，交给垃圾回收器自动回收。  
垃圾回收：将已经分配出去，但是不再使用的内存回收回来，为了能够再次分配。在jvm的语境下，垃圾指的是死亡的对象所占的堆空间。  
那么如何判断对象是死亡的还是存活的？  
- 引用计数法：为每个对象添加一个引用计数器，用来统计指向该对象的引用个数。当计数为0时代表对象死亡，但是该方法无法识别出来两个对象之间循环引用的情况，但是这两个对象其实已经没有其他的对象  
  引用，属于死亡对象，这样会造成内存泄露。    
- 可达性分析：目前jvm主流垃圾回收器采用可达性分析算法识别对象是否已死亡。  
可达性分析算法实质：将一些列的对象作为初始存活对象集合，称为GC Roots，然后从GC Roots触发，顺着对象引用关系，找所有能够被引用到的对象，然后把这些对象加入到存活对象集合中，这个过程称为mark标记  
过程。最终没有被标记到的对象，就是死亡对象，可以被回收。  
GC Roots一般包含：  
- java方法栈帧中的局部变量  
- 已加载类的静态变量  
- JNI handles
- 已经启动且没有停止的java线程  
有了可达性分析方法之后，还存在一些问题需要解决：  
比如在多线程环境下，其他线程可能会更新在可达性方法已经访问过的对象，更新了这个对象中的引用，会导致出现两种错误情况：
1. 之前标记为存活的对象，现在变成了死亡对象  
2. 之前判定为死亡的对象，现在变成了存活对象
第1种情况影响较小，这次垃圾回收只是少回收了部分对象。第2种情况影响就很大了，一旦原引用访问已经被回收的对象，就很有可能直接造成jvm崩溃。  
为了解决上面提到的两种错误情况，引入Stop-the-world和安全点  
### Stop-the-world和安全点safe point  
传统的垃圾回收算法采用了简单粗暴的方法Stop-the-world，即停止其他非垃圾回收线程的工作，直到垃圾回收过程完成。其他非垃圾回收线程等待垃圾回收的时间就是垃圾回收暂停时间GC pause。   
jvm的stop-the-world是通过安全点机制来实现的。当jvm收到垃圾回收器stop-the-world的请求时，会等待所有的线程都达到了安全点位置，才能让请求stop-the-world的线程进行独占工作。  
安全点的目的是：让线程达到一个稳定的执行状态，在这个状态下，jvm中的堆栈不会再发生变化，垃圾回收器就可以安全的进行可达性分析过程。  
比如：当java程序通过JNI执行本地代码时，如果这段本地代码不访问java对象，不调用java方法或者返回到原java方法，那么jvm的堆栈不会发生变化，这段代码就可以作为一个安全点。  
只要不离开这个安全点，jvm就能在垃圾回收的过程中，同时继续执行这段处于安全点的本地代码。  
由于本地代码需要通过JNI的API来完成：访问java对象、调用java方法、返回原java方法，所以可以在API的入口处进行安全点检测，检测一下是否有其他的线程请求工作线程停留在安全点中，在必要的时候  
可以挂起当前线程。  
java线程的几种状态：解释执行字节码、执行及时编译器生成的机器码、线程阻塞。  
线程阻塞属于安全点，其他两种状态属于运行时状态，需要jvm保证在可预见的时间内进入安全点，否则会造成垃圾回收线程一直在等待所有的线程进入安全点，从而造成垃圾回收的暂停时间变长。
- 解释执行字节码  
字节码和字节码之间都可以作为安全点，jvm在有安全点请求的之后，执行一条字节码就会进行一次安全点检测。  
- 执行及时编译生成的机器码  
执行生成的机器码比较复杂，这些代码直接运行在底层硬件上，不受jvm掌控，所以jvm及时编译器在生成机器码时，插入安全点检测。Hotspot jvm插入安全点检测的位置：  
    - 生成代码的方法出口处  
    - 非计数循环的循环回边（back-edge）  
为什么不在每条机器码的地方插入安全检测点呢？  
- 安全监测点本身也有一定的开销。Hotspot jvm已经将安全点检测简化成一个内存访问操作。在有安全点检测的情况，jvm会将安全点检测访问的内存所在页设置成不可读的，之后访问该内存会触发segfault，jvm  
  定义了一个segfault处理器，来截获这些触发了segfault的线程，然后挂起这些线程。  
- 及时编译器生成的机器码打乱了原本栈帧上的对象分布情况，在进入安全点之前，为了让垃圾回收器能找到GC Roots对象，机器码需要提供一些额外的信息，来表明哪些寄存器或者当前栈帧上哪些内存空间存放着对象引用，  
这些信息是需要不少空间来存储的  
基于上面的两个原因，所以即使编译器会避免插入过多的安全点检测。  
不同的即使编译器插入安全点的位置也可能不同，新一代的JIT编译器Graal，则还会在计数循环的循环回边处插入安全点检测。其他的虚拟机也可能选择在方法入口处插入安全点检测而不是在方法的出口处。  
选择性的插入安全点检测，是为了在可以接受的性能开销和内存开销之内，避免机器码长时间不进入安全点的情况，防止造成垃圾回收暂定时间过长的情况。  
### 垃圾回收的三种方式  
根据可达性方法、安全点检测、stop-the-world标记完所有存活的对象之后，就可以进行死亡对象的回收了，那主流的死亡对象的回收方式可分为三种：清除、压缩、复制。    
- 清除 sweep  
把死亡对象占的内存标记成空闲，并且记录到一个空闲列表 free list中。当需要新建对象时，内存管理模块会从空闲列表中找空闲内存，分配给新建的对象。  
清除方式简单，但是存在两个缺点：  
    1. 内存碎片：jvm堆中对象必须是连续分布的，可能会出现空闲内存总空间够用，但是没法分配成功的情况。  
    2. 内存分配效率低：空闲列表的方式，需要遍历空闲列表，找到可以满足分配条件的内存。和用于内存空间连续的指针加法 pointer bumping相比效率较低。    
- 压缩 compact
把存活对象聚集到内存区域的起始位置，这样可以让空闲内存是连续的。解决了复制方式的内存碎片问题和分配低效问题，但是带来了压缩性能开销的问题。  
- 拷贝 copy
把内存等分成两份，分别用两个指针from和to维护，每次分配内存都是从from指针指向的内存区域中分配。垃圾回收时，会将from中的存活对象复制到to中，然后交换from和to指针。  
拷贝方法解决了内存碎片问题、分配低效问题、压缩性能开销问题，但是缺点很明显：浪费一般的内存空间，牺牲了空间，换时间。  
当代的垃圾回收器往往会综合上述的几种回收方式，综合优点，避免缺点。  
假设大部分的java对象只存活一小段时间，而存活下来的小部分java对象则会存活很长一段时间，且这个假设一版情况下是成立的。  
根据这个假设，jvm提出了分代回收思想。将堆分为两部分，一部分称作年轻代，另一部分称为年老代。新生代用来存储新建的对象，当对象存活时间够长时，会被移动到老年代。  
jvm可以通过分代，然后给不同的代使用不同的垃圾回收算法。年轻代：假设的是大部分java对象只会存活很短的时间，那么就需要比较频繁的回收死亡对象，需要采用耗时比较短的垃圾回收算法，可以频繁快速的回收垃圾对象，让  
大部分垃圾对象可以在新生代被回收掉。  
对于老年代，假设大部分对象在年轻代被回收，只有少部分对象进入老年代，且这部分对象会较长时间存活下去。当这个假设错了，从年轻代到老年代有很多对象，当jvm堆内存空间不足时，就需要触发Full GC，jvm往往需要做一次  
全堆扫描，耗时会比较长。  
### jvm的堆划分  
jvm将堆划分成：新生代、老年代。  
- 新生代  
  新生代划分成：Eden区、Survivor区（两个大小相同的部分from、to）。默认情况下jvm采用一种动态分配的策略，动态调整Eden区和survivor区的比例，对应jvm配置参数  
  -XX:+UsePSAdaptiveSurvivorPolicy，根据生成对象的速度和Survivor区的使用情况动态调整。  
  也可以通过使用参数-XX:SurvivorRatio固定Eden和Survivor区大小比例。  
  当调用new指令时，会在Eden区划出来一块儿作为存储该对象的内存。因为堆是线程共享的，如果直接在堆上分配内存需要同步机制。jvm通过使用TLAB技术从Eden区为新建对象  
  分配内存。TLAB全称为 Tread Local allocation buffer，线程本地分配缓存。对应虚拟机参数-XX:+UseTLAB，默认开启。  
  使用TLAB技术具体过程：每个线程可以向jvm申请一段连续的内存，作为该线程私有的TLAB，这个操作需要加锁。  
  线程维护两个指针，一个指向TLAB的空闲内存的起始地址，另一个指向TLAB末尾。当前线程需要新建对象，调用new指令时，可以直接通过指针加法bump the pointer来实现，  
  把指向空闲内存起始位置的指针加上请求的字节数，如果增加之后的指针小于等于末尾指针，则代表分配成功；否则代表空间不够，需要重新向jvm申请一块儿TLAB。  
  > bump有increase的意思，提高，增加 比如升版本号 bump the version number
  
  当Eden区的空间用完了，jvm会触发一次minor gc，回收年轻代垃圾，存活下来的对象（Eden和from survivor中）会复制到to survivor区，然后交换from和to指针。  
  jvm会记录年轻代中对象从from复制到to survivor区的次数，超过一定阈值（对应jvm参数-XX:+MaxTenuringThreshold 默认是15），就会将该对象移动到老年代中。  
  在单个survivor区，如果使用的比例已经超过了一定值（50% 对应jvm参数 -XX:TargetSurvivorRatio），较高复制次数的对象也会被移动到老年代中。
  在年轻代使用的垃圾回收算法为 标记-复制算法，将survivor区中存活时间较长的对象移动到老年代中，把from survivor和eden中的存活对象复制到to survivor中。  
  minor gc不用对整个堆进行回收，这样能保证回收的效率。在遍历初始的GC Roots时，判断指向堆内存中对象在老年代中，可以跳过，避免遍历老年代中所有存活的对象，浪费时间，  
  如果GC Roots指向的是年轻代的对象，则继续遍历标记年轻代中存活对象。  
  这样存在一个问题：年老代中的对象可能会引用到年轻代中的对象，如果不考虑这一点，会造成部分年轻代  对象漏标记成存活对象了。那么怎么找到这种跨代引用？  
  - 遍历整个年老代中的对象，判断是否直接引用了年轻代中的对象。问题：效率太低。  
  - 遍历GC Roots时，遍历年老代中所有存活的对象，一直找到底，标记年轻代中被老年代中对象引用的对象。问题：一般在年老代中存活的对象，很多可能会长时间存在，那么数量也会相对较多，  
  这样的标记效率还是低。  
  - Hotspot解决方案：卡表技术 Card Table。将整个堆划分成一个个大小为512字节的卡，对应维护一个卡表，卡表中的每一条记录为一个标示，代表对应的卡是否可能存在引用年轻代的对象，  
  如果可能存在，则对应的卡是脏的。在进行minor gc时，不需要遍历整个堆，只需要遍历卡表，遇到脏的卡，就把卡中对应的对象加入GC Roots中。遍历完所有的卡之后，jvm将所有脏卡标示为不脏清零。  
  进行minor gc过程中，会复制存活的对象到to survivor中，对象换地儿了，需要更新指向该对象的引用，在更新对象引用时，jvm把引用该对象的对象所在卡标记成脏的。  
  如果要保证每个可能包含了指向新生代对象的卡都被标记成脏卡，需要jvm截获每个引用型实例变量的写操作，并修改对应卡是否脏的标示。  
  及时编译器生成的代码，需要插入写屏障 write barrier，jvm为了保证写屏障代码简洁，引用不管是不是修改成指向年轻代的对象，都会把对应的卡标记成脏的。    
  `CARD_TABLE[this address >> 9] = DIRTY`  
  写屏障会带来一定的开销，但是缩短了gc停顿时间，提高了minor gc的吞吐率（应用代码运行时间/(应用代码运行时间+垃圾回收时间)）。 存在的问题：可能在高并发环境下，写屏障带来  
  伪共享（false sharing）的问题。在hotspot中，卡表是通过byte数组实现，一个64字节的缓存行，可以保存64个卡表标示，每个卡对应512字节，则对应32KB的内存。如果存在两个java线程同时修改  
  这个32KB的内存中的引用，会造成存在存在同一个卡表缓存行需要写回、无效化、同步操作，间接影响程序性能。  
  Hotspot引入参数-XX:+UseCondCardMark，减少写卡表的操作次数，如果卡表对应标示已经是DIRTY的，就不再更新。  
- 老年代   
## 13 java内存模型  








  
   
   
 




































  






                


    
        
       
    