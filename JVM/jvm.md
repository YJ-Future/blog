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

## JVM如何实现反射  
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





                


    
        
       
    