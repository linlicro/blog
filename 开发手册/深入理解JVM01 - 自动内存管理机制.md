# 深入理解JVM01 - 自动内存管理机制

## 内存模型

下图是描述JVM内存模型的图:

![](media/15475103362289/15475447681280.png)

JVM包含两个子系统和两个组件，两个子系统为Class loader(类装载)、Execution engine(执行引擎)；两个组件为Runtime data area(运行时数据区)、Native Interface(本地接口)。

Class loader(类装载): 根据给定的全限定名类名(如：java.lang.Object)来装载class文件到Runtime data area中的method area。

Execution engine（执行引擎）: 执行classes中的指令。

Native Interface(本地接口): 与native libraries交互，是其它编程语言交互的接口。

Runtime data area(运行时数据区): 这就是我们常说的JVM的内存。

* 程序计数器（Program Counter Register）: 一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。
* Java虚拟机栈（Java Virtual Machine Stacks）: 是Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame[1]）用于存储局部变量表、操作数栈、动态链接、方法出口等信息。
* 本地方法栈（Native Method Stack）: 与虚拟机栈所发挥的作用是非常相似的，为虚拟机使用到的Native方法服务。
* Java堆（Java Heap）: 是Java虚拟机所管理的内存中最大的一块，唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。
* 方法区（Method Area）: 用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。
* 运行时常量池（Runtime Constant Pool）: 是方法区的一部分，Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。
* 直接内存（Direct Memory）: 并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。JDK 1.4中新加入了NIO（New Input/Output）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。

Heap和Method Area是被所有线程的共享使用的；而Java stack, Program counter 和Native method stack是以线程为粒度的，每个线程独自拥有自己的部分。

## 对象探秘

#### 创建

下面是对象创建的主要流程:

![JVM对象创建](media/15475103362289/JVM%E5%AF%B9%E8%B1%A1%E5%88%9B%E5%BB%BA.png)

虚拟机遇到一条new指令时，先检查常量池是否已经加载相应的类，如果没有，必须先执行相应的类加载。类加载通过后，接下来分配内存。若Java堆中内存是绝对规整的，使用“指针碰撞“方式分配内存；如果不是规整的，就从空闲列表中分配，叫做”空闲列表“方式。划分内存时还需要考虑一个问题-并发，也有两种方式: CAS同步处理，或者本地线程分配缓冲(Thread Local Allocation Buffer, TLAB)。然后内存空间初始化操作，接着是做一些必要的对象设置(元信息、哈希吗...)，最后执行<init>方法。

#### 内存布局

一个Java对象在内存中包括对象头、实例数据和补齐填充3个部分：

![对象的内存布局](media/15475103362289/%E5%AF%B9%E8%B1%A1%E7%9A%84%E5%86%85%E5%AD%98%E5%B8%83%E5%B1%80.png)

对象头:

* Mark Word：包含一系列的标记位，比如轻量级锁的标记位，偏向锁标记位等等。在32位系统占4字节，在64位系统中占8字节；
* Class Pointer：用来指向对象对应的Class对象（其对应的元数据对象）的内存地址。在32位系统占4字节，在64位系统中占8字节；
* Length：如果是数组对象，还有一个保存数组长度的空间，占4个字节；

对象实际数据: 

对象实际数据包括了对象的所有成员变量，其大小由各个成员变量的大小决定，比如：byte和boolean是1个字节，short和char是2个字节，int和float是4个字节，long和double是8个字节，reference是4个字节（64位系统中是8个字节）。

对齐填充:

Java对象占用空间是8字节对齐的，即所有Java对象占用bytes数必须是8的倍数。例如，一个包含两个属性的对象：int和byte，这个对象需要占用8+4+1=13个字节，这时就需要加上大小为3字节的padding进行8字节对齐，最终占用大小为16个字节。

#### 访问定位

目前主流的访问方式有通过句柄和直接指针两种方式。

![访问定位](media/15475103362289/%E8%AE%BF%E9%97%AE%E5%AE%9A%E4%BD%8D.png)

这两种访问方式各有利弊，使用句柄访最大的好处是reference中存储着稳定的句柄地址，当对象移动之后（垃圾收集时移动对象是非常普遍的行为），只需要改变句柄中的对象实例地址即可，reference不用修改。 

使用指针访问的好处是访问速度快，它减少了一次指针定位的时间开销，由于java是面向对象的语言，在开发中java对象的访问非常的频繁，因此这类开销积少成多也是非常可观的，反之则提升访问速度。对于HotSpot虚拟机来说，使用的就是直接指针访问的方式。


## 实战: OutOfMemoryError异常

#### Java 堆溢出

Java堆用来存储对象，因此只要不断创建对象，并保证 GC Roots 到对象之间有可达路径来避免垃圾回收机制清楚这些对象，那么当对象数量达到最大堆容量时就会产生 OOM。

```java
/**
 * java堆内存溢出测试
 * VM Args: -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {

    static class OOMObject{}

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        while (true) {
            list.add(new OOMObject());
        }
    }
}
```

运行结果：

```
java.lang.OutOfMemoryError: Java heap space 
Dumping heap to java_pid7164.hprof … 
Heap dump file created [27880921 bytes in 0.193 secs] 
Exception in thread “main” java.lang.OutOfMemoryError: Java heap space 
at java.util.Arrays.copyOf(Arrays.java:2245) 
at java.util.Arrays.copyOf(Arrays.java:2219) 
at java.util.ArrayList.grow(ArrayList.java:242) 
at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:216) 
at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:208) 
at java.util.ArrayList.add(ArrayList.java:440) 
at com.jvm.oom.HeapOOM.main(HeapOOM.java:17)
```

堆内存 OOM 是经常会出现的问题，异常信息会进一步提示 Java heap space

#### 虚拟机栈和本地方法栈溢出

在 HotSpot 虚拟机中不区分虚拟机栈和本地方法栈，栈容量只由 -Xss 参数设定。关于虚拟机栈和本地方法栈，在 Java 虚拟机规范中描述了两种异常：

* 如果线程请求的栈深度大于虚拟机所允许的最大深度，将抛出 StackOverflowError 异常。
* 如果虚拟机在扩展栈时无法申请到足够的内存空间，则抛出 OutOfMemoryError 异常。

```java
/**
 * 虚拟机栈和本地方法栈内存溢出测试，抛出stackoverflow exception
 * VM ARGS: -Xss128k 减少栈内存容量
 */
public class JavaVMStackSOF {

    private int stackLength = 1;

    public void stackLeak () {
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) throws Throwable {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length = " + oom.stackLength);
            throw e;
        }

    }

}
```

运行结果:

```
stack length = 11420 
Exception in thread “main” java.lang.StackOverflowError 
at com.jvm.oom.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:12) 
at com.jvm.oom.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:13) 
at com.jvm.oom.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:13) 
```

以上代码在单线程环境下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配时，抛出的都是 StackOverflowError 异常。 

如果测试环境是多线程环境，通过不断建立线程的方式可以产生内存溢出异常，代码如下所示。但是这样产生的 OOM 与栈空间是否足够大不存在任何联系，在这种情况下，为每个线程的栈分配的内存足够大，反而越容易产生OOM 异常。这点不难理解，每个线程分配到的栈容量越大，可以建立的线程数就变少，建立多线程时就越容易把剩下的内存耗尽。**这点在开发多线程的应用时要特别注意。如果建立过多线程导致内存溢出，在不能减少线程数或更换64位虚拟机的情况下，只能通过减少最大堆和减少栈容量来换取更多的线程。**

```java
/**
 * JVM 虚拟机栈内存溢出测试, 注意在windows平台运行时可能会导致操作系统假死
 * VM Args: -Xss2M -XX:+HeapDumpOnOutOfMemoryError
 */

public class JVMStackOOM {

    private void dontStop() {
        while (true) {}
    }

    public void stackLeakByThread() {
        while (true) {
            Thread thread = new Thread(new Runnable() {

                @Override
                public void run() {
                    dontStop();
                }
            });
            thread.start();
        }
    }

    public static void main(String[] args) {
        JVMStackOOM oom = new JVMStackOOM();
        oom.stackLeakByThread();
    }
}
```

#### 方法区和运行时常量池溢出

方法区用于存放Class的相关信息，对这个区域的测试，基本思路是运行时产生大量的类去填满方法区，直到溢出。使用CGLib实现。

方法区溢出也是一种常见的内存溢出异常，在经常生成大量Class的应用中，需要特别注意类的回收情况，这类场景除了使用了CGLib字节码增强和动态语言外，常见的还有JSP文件的应用(JSP第一次运行时要编译为Java类)、基于OSGI的应用等。

```java
/**
 * 测试JVM方法区内存溢出
 * VM Args: -XX:PermSize=10M -XX:MaxPermSize=10M
 */
public class MethodAreaOOM {

    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                @Override
                public Object intercept(Object obj, Method method, Object[] args,
                        MethodProxy proxy) throws Throwable {
                    return proxy.invokeSuper(obj, args);
                }
            });
            enhancer.create();
        }
    }

    static class OOMObject{}
}
```

#### 本机直接内存溢出

DirectMemory 容量可通过 -XX:MaxDirectMemorySize 指定，如不指定，则默认与Java堆最大值一样。测试代码使用了 Unsafe 实例进行内存分配。

由 DirectMemory 导致的内存溢出，一个明显的特征是在Heap Dump 文件中不会看见明显的异常，如果发现 OOM 之后 Dump 文件很小，而程序直接或间接使用了NIO，那就可以考虑检查一下是不是这方面的原因。

```
/**
 * 测试本地直接内存溢出
 * VM Args: -Xmx20M -XX:MaxDirectMemorySize=10M
 * @author linli.cro
 */
public class DirectMemoryOOM {

    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws Exception {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe = (Unsafe) unsafeField.get(null);
        while (true) {
            unsafe.allocateMemory(_1MB);
        }
    }
}
```

## 参考

* 《深入理解Java虚拟机: JVM高级特性与最佳实践》


