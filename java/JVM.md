###JVM的系统结构
	JVM主要包括两个子系统和两个组件。   
		- 两个子系统分别是Class loader子系统和Execution engine(执行引擎) 子系统；  
			* Class loader子系统的作用：根据给定的全限定名类名(如 java.lang.Object)来装载class文件的内容到 Runtime data area中的method area(方法区域)。Java程序员可以extends java.lang.ClassLoader类来写自己的Class loader。
			* Execution engine子系统的作用：执行classes中的指令。任何JVM specification实现(JDK)的核心都是Execution engine，不同的JDK例如Sun 的JDK 和IBM的JDK好坏主要就取决于他们各自实现的Execution engine的好坏     
		- 两个组件分别是Runtime data area (运行时数据区域)组件和Native interface(本地接口)组件。  
			* Native interface组件：与native libraries交互，是其它编程语言交互的接口。当调用native方法的时候，就进入了一个全新的并且不再受虚拟机限制的世界，所以也很容易出现JVM无法控制的native heap OutOfMemory。
			* Runtime Data Area组件：这就是我们常说的JVM的内存了。   
			它主要分为五个部分
			   	*  Heap (堆)：一个Java虚拟实例中只存在一个堆空间   
				* Method Area(方法区域)：被装载的class的信息存储在Method area的内存中。  当虚拟机装载某个类型时，它使用类装载器定位相应的class文件，然后读入这个    class文件内容并把它传输到虚拟机中。  
				* Java Stack(java的栈)：虚拟机只会直接对Java stack执行两种操作：以帧为单位的压栈或出栈   
				* Program Counter(程序计数器)：每一个线程都有它自己的PC寄存器，也是该线程启动时创建的。PC寄存器的内容总是指向下一条将被执行指令的地址，这里的地址可以是一个本地指针，也可以是在方法区中相对应于该方法起始指令的偏移量。    
				* Native method stack(本地方法栈)：保存native方法进入区域的地址
		注意：以上五部分只有Heap 和Method Area是被所有线程的共享使用的；而Java stack, Program counter 和Native method stack是以线程为粒度的，每个线程独自拥有自己的部分. 
       
###JVM内存回收
Sun（现在应该叫oracle）的JVM Generational Collecting(垃圾回收)原理是这样的：把对象分为年青代(Young)、年老代(Tenured)、持久代(Perm)，三类 对不同生命周期的对象使用不同的算法。(基于对对象生命周期分析)

	- Young（年轻代）
	年轻代分三个区。一个Eden区，两个Survivor区。大部分对象在Eden区中生成。当Eden区满时，还存活的对象将被复制到Survivor区（两个中的一个），当这个Survivor区满时，此区的存活对象将被复制到另外一个Survivor区，当这个Survivor去也满了的时候，从第一个Survivor区复制过来的并且此时还存活的对象，将被复制年老区(Tenured。需要注意，Survivor的两个区是对称的，没先后关系，所以同一个区中可能同时存在从Eden复制过来 对象，和从前一个Survivor复制过来的对象，而复制到年老区的只有从第一个Survivor去过来的对象。而且，Survivor区总有一个是空的。  
	- Tenured（年老代）
	年老代存放从年轻代存活的对象。一般来说年老代存放的都是生命期较长的对象。  
	- Perm（持久代）
	用于存放静态文件，如今Java类、方法等。持久代对垃圾回收没有显著影响，但是有些应用可能动态生成或者调用一些class，例如Hibernate等，在这种时候需要设置一个比较大的持久代空间来存放这些运行过程中新增的类。持久代大小通过-XX:MaxPermSize=进行设置。

	举个例子：当在程序中生成对象时，正常对象会在年轻代中分配空间，如果是过大的对象也可能会直接在年老代生成（据观测在运行某程序时候每次会生成一个十兆的空间用收发消息，这部分内存就会直接在年老代分配）。年轻代在空间被分配完的时候就会发起内存回收，大部分内存会被回收，一部分幸存的内存会被拷贝至Survivor的from区，经过多次回收以后如果from区内存也分配完毕，就会也发生内存回收然后将剩余的对象拷贝至to区。等到to区也满的时候，就会再次发生内存回收然后把幸存的对象拷贝至年老区。  
		
	通常我们说的JVM内存回收总是在指堆内存回收，确实只有堆中的内容是动态申请分配的，所以以上对象的年轻代和年老代都是指的JVM的Heap空间，而持久代则是之前提到的Method Area，不属于Heap。     

###内存管理的一些建议
	1、手动将生成的无用对象，中间对象置为null，加快内存回收。
	2、对象池技术 如果生成的对象是可重用的对象，只是其中的属性不同时，可以考虑采用对象池来较少对象的生成。如果有空闲的对象就从对象池中取出使用，没有再生成新的对象，大大提高了对象的复用率。
	3、JVM调优 通过配置JVM的参数来提高垃圾回收的速度，如果在没有出现内存泄露且上面两种办法都不能保证内存的回收时，可以考虑采用JVM调优的方式来解决，不过一定要经过实体机的长期测试，因为不同的参数可能引起不同的效果。如-Xnoclassgc参数等。  

###推荐的两款内存检测工具
	1、jconsole JDK自带的内存监测工具，路径jdk bin目录下jconsole.exe，双击可运行。连接方式有两种，第一种是本地方式如调试时运行的进程可以直接连，第二种是远程方式，可以连接以服务形式启动的进程。远程连接方式是：在目标进程的jvm启动参数中添加-Dcom.sun.management.jmxremote.port=1090 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false 1090是监听的端口号具体使用时要进行修改，然后使用IP加端口号连接即可。通过该工具可以监测到当时内存的大小，CPU的使用量以及类的加载，还提供了手动gc的功能。优点是效率高，速度快，在不影响进行运行的情况下监测产品的运行。缺点是无法看到类或者对象之类的具体信息。使用方式很简单点击几下就可以知道功能如何了，确实有不明白之处可以上网查询文档。

	2、JProfiler 收费的工具，但是到处都有破解办法。安装好以后按照配置调试的方式配置好一个本地的session即可运行。可以监测当时的内存、CPU、线程等，能具体的列出内存的占用情况，还可以就某个类进行分析。优点很多，缺点太影响速度，而且有的类可能无法被织入方法，例如我使用jprofiler时一直没有备份成功过，总会有一些类的错误

###一些简单概念
	1.堆(Heap)和非堆(Non-heap)内存
	按照官方的说法：“Java 虚拟机具有一个堆，堆是运行时数据区域，所有类实例和数组的内存均从此处分配。堆是在 Java 虚拟机启动时创建的。”“在JVM中堆之外的内存称为非堆内存(Non-heap memory)”。可以看出JVM主要管理两种类型的内存：堆和非堆。简单来说堆就是Java代码可及的内存，是留给开发人员使用的；非堆就是JVM留给 自己用的，所以方法区、JVM内部处理或优化所需的内存(如JIT编译后的代码缓存)、每个类结构(如运行时常数池、字段和方法数据)以及方法和构造方法 的代码都在非堆内存中。
		
	2.堆内存分配
	JVM初始分配的内存由-Xms指定，默认是物理内存的1/64；JVM最大分配的内存由 -Xmx指定，默认是物理内存的1/4。默认空余堆内存小于40%时，JVM就会增大堆直到-Xmx的最大限制；空余堆内存大于70%时，JVM会减少堆 直到-Xms的最小限制。因此服务器一般设置-Xms、-Xmx相等以避免在每次GC 后调整堆的大小。

	3.非堆内存分配
	JVM使用-XX:PermSize设置非堆内存初始值，默认是物理内存的1/64；由XX:MaxPermSize设置最大非堆内存的大小，默认是物理内存的1/4。

	4.JVM内存限制(最大值)
	首先JVM内存限制于实际的最大物理内存(废话！呵呵)，假设物理内存无限 大的话，JVM内存的最大值跟操作系统有很大的关系。简单的说就32位处理器虽然可控内存空间有4GB,但是具体的操作系统会给一个限制，这个限制一般是 2GB-3GB（一般来说Windows系统下为1.5G-2G，Linux系统下为2G-3G），而64bit以上的处理器就不会有限制了

###Java语言具备GC(垃圾回收)的能力，内存管理不需要应用程序去过问，这很方便。但是，GC是怎么进行的，JVM的内存参数应该怎么调整，如何优化

	1.当JVM进行GC的时候，是要消耗CPU资源和需要一定时间的，这会影响到程序的正常运行，因此需要尽可能减少GC消耗的时间。Java程序运行过程中，对象的生命周期有长有短，其中相当大部分是都是比较短命的，例如局部的对象一用完就可以回收了。在大多数情况下，只要能够及时回收这些短命对象的内存，就能够确保JVM有足够内存来分配给新的对象。因此JVM采用一种分代回收(generational collection) 的策略，用较高的频率对年轻的对象(young generation)进行扫描和回收，这种叫做minor collection，而对老对象(old generation)的检查回收频率要低很多，称为major collection。这样就不需要每次GC都将内存中所有对象都检查一遍。  
	总结：对象生命周期有唱有短 分代 分频率回收 

	2.Sun JVM 1.3 有两种最基本的内存收集方式：
	一种称为copying或scavenge，将所有仍然生存的对象搬到另外一块内存后，整块内存就可回收。这种方法有效率，但需要有一定的空闲内存，拷贝也有开销。这种方法用于minor collection。
	另外一种称为mark-compact，将活着的对象标记出来，然后搬迁到一起连成大块的内存，其他内存就可以回收了。这种方法不需要占用额外的空间，但速度相对慢一些。这种方法用于major collection.
	总结：搬运 和 标记 占用内存空间的大小有不一样

	3.JVM启动后，保留一段地址空间，这个空间的大小由-Xmx指定。这块空间的大小就是heap可能的最大值，但一开始不一定全都分配了物理内存，初始分配的heap大小由-Xms指定，如果-Xms小于-Xmx，剩余部分是virtual的，当需要的时候，再向OS申请。而且申请之后，是继续占用而不释放给该jvm以外的程序。比如你的jvm申请了1G的内存，刚开始用了200M，然后随着程序的进行，内存用到900M，然后进行垃圾回收，想释放一些内存给其他程序，这是不可以的，此时，jvm依然会保有着900M内存。
	总结：有点类似线程池的概念

	4.eden空间不够了就要进行minor collection，old generation空间不够要进行major collection，permanent generation空间不足会引发full GC。

	5.很多参数会影响里面各部分空间的分配。
	-XX:MinHeapFreeRatio与-XX:MaxHeapFreeRatio设定空闲内存占总内存的比例范围，这两个参数会影响GC的频率和单次GC的耗时。
	-XX:NewRatio决定young与old generation的比例。Young generation空间越大，minor collection频率越低，但是old generation空间小了，又可能导致major collection频率增加。
	-XX:NewSize和-XX:MaxNewSize直接指定了young generation的缺省大小和最大大小。

###心得分享
	1、jvm的内存分区分级大粒度管理相较memcache的固定单元小粒度内存管理，拥有更高的内存利用率，但带来内存碎片的问题。 
	2、为了解决内存碎片问题，jvm采取了碎片整理的方式，但碎片整理是很耗时的。
	3、为了提高碎片整理的效率，因此引入了周期性的GC，而且分区分级的方式也控制了每次GC和碎片整理的范围。 
	4、由于jvm使用堆内存来存储局部变量，而局部变量具有生存周期短，先申请的后释放的特点，因此在低级别的分区中进行GC是效率最高的方式。 感觉环环相扣，有点奇妙。
	再补充GC的一个作用：寻找回路的孤立存储，并释放其占用的空间。这更要求GC的非实时、周期性

###[GC](http://www.cnblogs.com/xrq730/p/4836700.html)
	* 如何确定某个对象是“垃圾”
	(1)引用计数法
		给对象中添加一个引用计数器，每当一个地方引用这个对象时，计数器值+1；当引用失效时，计数器值-1。
		任何时刻计数值为0的对象就是不可能再被使用的.
		这种算法很难解决对象之间相互引用的情况
	(2)可达性分析法
		这个算法的基本思想是通过一系列称为“GC Roots”的对象作为起始点，从这些节点向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链（即GC Roots到对象不可达）时，则证明此对象是不可用的。在Java语言中可以作为GC Roots的对象包括：
		· 虚拟机栈中引用的对象
		· 方法区中静态属性引用的对象
		· 方法区中常量引用的对象
		· 本地方法栈中JNI（即Native方法）引用的对象

	* 垃圾收集算法
	(1)标记-清除（Mark-Sweep）算法
		分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，标记完成后统一回收所有被标记的对象
		这种算法的不足主要体现在效率和空间，从效率的角度讲，标记和清除两个过程的效率都不高;
		从空间的角度讲，标记清除后会产生大量不连续的内存碎片，内存碎片太多可能会导致以后程序运行过程中在需要分配较大对象时，无法找到足够的连续内存而不得不提前触发一次垃圾收集动作
	(2)复制（Copying）算法
		复制算法是为了解决效率问题而出现的，它将可用的内存分为两块，每次只用其中一块，当这一块内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已经使用过的内存空间一次性清理掉。
		这样每次只需要对整个半区进行内存回收，内存分配时也不需要考虑内存碎片等复杂情况，只需要移动指针，按照顺序分配即可
		不过这种算法有个缺点，内存缩小为了原来的一半，这样代价太高了
	(3)标记-整理（Mark-Compact）算法
		过程与标记-清除算法一样，不过不是直接对可回收对象进行清理，而是让所有存活对象都向一端移动，然后直接清理掉边界以外的内存
	(4)分代收集
		现代商用虚拟机基本都采用分代收集算法来进行垃圾回收。这种算法没什么特别的，无非是上面内容的结合罢了，根据对象的生命周期的不同将内存划分为几块，然后根据各块的特点采用最适当的收集算法。大批对象死去、少量对象存活的，使用复制算法，复制成本低；对象存活率高、没有额外空间进行分配担保的，采用标记-清理算法或者标记-整理算法

	* 垃圾收集器
	(1)Serial收集器
		采用复制算法的单线程的收集器，单线程一方面意味着它只会使用一个CPU或一条线程去完成垃圾收集工作，另一方面也意味着它进行垃圾收集时必须暂停其他线程的所有工作，直到它收集结束为止
	(2)ParNew收集器
		ParNew收集器其实就是Serial收集器的多线程版本，除了使用多条线程进行垃圾收集外，其余行为和Serial收集器完全一样，包括使用的也是复制算法
	(3)Parallel收集器
		Parallel收集器也是一个新生代收集器，也是用复制算法的收集器，也是并行的多线程收集器.
		CMS等收集器的关注点是尽可能缩短垃圾收集时用户线程的停顿时间，而Parallel收集器的目标则是打到一个可控制的吞吐量.
		Parallel收集器是虚拟机运行在Server模式下的默认垃圾收集器.
	(4)CMS收集器
		CMS收集器是一种以获取最短回收停顿时间为目标的老年代收集器.
	(5)G1收集器

	* 4种引用状态
	(1)强引用
		代码中普遍存在的类似"Object obj = new Object()"这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象
	(2)软引用
		描述有些还有用但并非必需的对象。在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围进行二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常.Java中的类SoftReference表示软引用
	(3)弱引用
		描述非必需对象。被弱引用关联的对象只能生存到下一次垃圾回收之前，垃圾收集器工作之后，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。
		Java中的类WeakReference表示弱引用
	(4)虚引用
		这个引用存在的唯一目的就是在这个对象被收集器回收时收到一个系统通知，被虚引用关联的对象，和其生存时间完全没关系。Java中的类PhantomReference表示虚引用

	* Minor GC,Major GC,Full GC
		Minor GC：指发生在新生代的垃圾收集动作，因为 Java 对象大多都具备朝生夕灭的特性，所以 Minor GC 非常频繁，一般回收速度也比较快。
		Major GC/Full GC ：指发生在老年代的 GC，出现了 Major GC，经常会伴随至少一次的 Minor GC
		Full GC:针对老年代/永久代进行GC,stop the world

###JVM系统结构
	堆(Heap):jvm中内存占用最大的一块,是所有线程共享的一块内存区域.在jvm启动时创建,存放的是所有对象实例（或数组），所有的对象实例都在这里进行动态分配，当类空间无法再扩张会抛出OutOfMemoryError异常。Java堆是垃圾收集器管理的主要区域，而收集器采用分代收集算法。

	方法区(Method Area)：与堆类似,也是各个线程共享的内存区域，主要用来存储加载的类信息，常量，静态变量，即时编译器编译后的代码等数据，当方法区无法满足内存分配时,也抛出OutOfMemoryError异常。运行时常量池是方法区的一部分，用于存放编译期生成的各种字面量和符号引用相对于Class文件常量池的重要特征是具备动态性（常量并非强制编译期产生，运行期间也可以新增，例如String类的intern()方法）。

	虚拟机栈：和程序计数器一样，都属于线程私有,生命周期与线程相同，描述的是java方法执行的内存模型,每个方法执行都会创建一个栈帧，用于存储局部变量表，操作栈，动态链接，方法出口等信息，每一个方法被调用直至执行完成的过程，就对应一个栈帧在jvm stack 从入栈到出栈的过程.局部变量表存放了编译期可知的各种数据基本类型(Boolean,byte,char,short,int,float,long,double),以及对象的引用。这个区域中定义了2种异常情况,如果线程请求的栈深度大于jvm所允许的深度,将抛出StackOverflowError异常,如果jvm可以动态扩张，当扩张无法申请到足够的内存空间是会抛出OutOfMemoryError异常。（这些数据区域异常将在下面的例子都讲到）。

	本地方法栈：与虚拟机栈比较相似。其区别：虚拟机栈为虚拟机执行Java方法服务，而本地方法栈则为虚拟机使用Native方法服务。

	程序计数器(Program Counter Register):属于线程私有的，占用的内存空间较少，可以看成是当前线程所执行字节码的行号指示器，字节码解释器工作时就是通过改变这个计数器的值来选择下一条，需要执行的字节码指令，分支，循环，跳转，异常处理，线程恢复等基础功能需要依赖这个计数器来完成，这个区域是jvm规范中没有规定任何OutOfMemoryError情况区域。

###为什么程序计数器要线程私有
	由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）只会执行一条线程中的指令。
	因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间的计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。 

###[对象访问方式](http://blog.csdn.net/java2000_wl/article/details/8015105)
	由于reference类型在Java虚拟机规范里面只规定了一个指向对象的引用，并没有定义这个引用应该通过哪种方式去定位，以及访问到Java堆中的对
	象的具体位置，因此不同虚拟机实现的对象访问方式会有所不同，主流的访问方式有两种：使用句柄和直接指针。 如果使用句柄访问方式，Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据和类型数据各自的具体地址信息，

	这两种对象的访问方式各有优势，使用句柄访问方式的最大好处就是reference中存储的是稳定的句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要被修改。使用直接指针访问方式的最大好处就是速度更快，它节省了一次指针定位的时间开销，由于对象的访问在Java中非常频繁，因此这类开销积少成多后也是一项非常可观的执行成本。   

###什么叫java的内存泄露
	在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是可达的，即在有向图中，存在通路可以与其相连（也就是说仍存在该内存对象的引用）；
	其次，这些对象是无用的，即程序以后不会再使用这些对象。
	如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

	Java内存泄露的原因
		1、静态集合类像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，所有的对象Object也不能被释放，因为他们也将一直被Vector等应用着。
		2、内部类和外部类的引用容易出现内存泄露的问题
		3、监听器的使用，java中往往会使用到监听器，在释放对象的同时没有相应删除监听器的时候也可能导致内存泄露。
		4、大量临时变量的使用，没有及时将对象设置为null也可能导致内存的泄露
		5、数据库的连接没有关闭情况，包括连接池方法连接数据库，如果没有关闭ResultSet等也都可能出现内存泄露的问题。

###堆和栈的优缺点
	堆的优势是可以动态分配内存大小，生存期也不必事先告诉编译器，因为它是在运行时动态分配内存的。
	缺点就是要在运行时动态分配内存，存取速度较慢； 
	栈的优势是，存取速度比堆要快，仅次于直接位于CPU中的寄存器。
	但缺点是，存在栈中的数据大小与生存期必须是确定的，缺乏灵活性。

###对象
	在Java中，创建一个对象包括对象的声明和实例化两步，下面用一个例题来说明对象的内存模型。　　假设有类Rectangle定义如下：
	public class Rectangle {
		double width;
		double height;
		public Rectangle(double w,double h){
			w = width;
			h = height;
		}
	} 
	(1)声明对象时的内存模型
	用Rectanglerect；声明一个对象rect时，将在栈内存为对象的引用变量rect分配内存空间，但Rectangle的值为空，称rect是一个空对象。空对象不能使用，因为它还没有引用任何"实体"。
	(2)对象实例化时的内存模型
	当执行rect=newRectangle(3,5)；时，会做两件事：　在堆内存中为类的成员变量width,height分配内存，并将其初始化为各数据类型的默认值；接着进行显式初始化（类定义时的初始化值）；最后调用构造方法，为成员变量赋值。  返回堆内存中对象的引用（相当于首地址）给引用变量rect,以后就可以通过rect来引用堆内存中的对象了。

	一个类通过使用new运算符可以创建多个不同的对象实例，这些对象实例将在堆中被分配不同的内存空间，改变其中一个对象的状态不会影响其他对象的状态。

	Rectangle r1= new Rectangle(3,5);
	Rectangle r2=r1;
	则在堆内存中只创建了一个对象实例，在栈内存中创建了两个对象引用，两个对象引用同时指向一个对象实例。
###静态变量
	用static的修饰的变量和方法，实际上是指定了这些变量和方法在内存中的"固定位置"－static storage，可以理解为所有实例对象共有的内存空间。
	static变量有点类似于C中的全局变量的概念；静态表示的是内存的共享，就是它的每一个实例都指向同一个内存地址。
	把static拿来，就是告诉JVM它是静态的，它的引用（含间接引用）都是指向同一个位置，在那个地方，你把它改了，它就不会变成原样，你把它清理了，它就不会回来了。         那静态变量与方法是在什么时候初始化的呢？对于两种不同的类属性，static属性与instance属性，初始化的时机是不同的。
	instance属性在创建实例的时候初始化，static属性在类加载，也就是第一次用到这个类的时候初始化，对于后来的实例的创建，不再次进行初始化。我们常可看到类似以下的例子来说明这个问题：
	class Student{
	static int numberOfStudents=0;
	Student()
	{
	numberOfStudents++;
	}
	}
	每一次创建一个新的Student实例时,成员numberOfStudents都会不断的递增,并且所有的Student实例都访问同一个numberOfStudents变量,实际上int numberOfStudents变量在内存中只存储在一个位置上。

###垃圾回收机制八问
	（问题一：什么叫垃圾回收机制？） 垃圾回收是一种动态存储管理技术，它自动地释放不再被程序引用的对象，按照特定的垃圾收集算法来实现资源自动回收的功能。
	当一个对象不再被引用的时候，内存回收它占领的空间，以便空间被后来的新对象使用，以免造成内存泄露。 
	
	（问题二：java的垃圾回收有什么特点？） JAVA语言不允许程序员直接控制内存空间的使用。内存空间的分配和回收都是由JRE负责在后台自动进行的，尤其是无用内存空间的回收操作(garbagecollection,也称垃圾回收)，只能由运行环境提供的一个超级线程进行监测和控制。 
	
	（问题三：垃圾回收器什么时候会运行？） 
	一般是在CPU空闲或空间不足时自动进行垃圾回收，而程序员无法精确控制垃圾回收的时机和顺序等。 
	
	（问题四：什么样的对象符合垃圾回收条件？） 
	当没有任何获得线程能访问一个对象时，该对象就符合垃圾回收条件。 
	
	（问题五：垃圾回收器是怎样工作的？） 垃圾回收器如发现一个对象不能被任何活线程访问时，他将认为该对象符合删除条件，就将其加入回收队列，但不是立即销毁对象，何时销毁并释放内存是无法预知的。垃圾回收不能强制执行，然而Java提供了一些方法（如：System.gc()方法），允许你请求JVM执行垃圾回收，而不是要求，虚拟机会尽其所能满足请求，但是不能保证JVM从内存中删除所有不用的对象。 
	
	（问题六：一个java程序能够耗尽内存吗？） 可以。垃圾收集系统尝试在对象不被使用时把他们从内存中删除。然而，如果保持太多活的对象，系统则可能会耗尽内存。垃圾回收器不能保证有足够的内存，只能保证可用内存尽可能的得到高效的管理。 
	
	（问题七：如何显示的使对象符合垃圾回收条件？） 
		（1） 空引用 ：当对象没有对他可到达引用时，他就符合垃圾回收的条件。也就是说如果没有对他的引用，删除对象的引用就可以达到目的，因此我们可以把引用变量设置为null，来符合垃圾回收的条件。
		StringBuffer sb = new StringBuffer("hello");
		System.out.println(sb);
		sb=null;
		（2） 重新为引用变量赋值：可以通过设置引用变量引用另一个对象来解除该引用变量与一个对象间的引用关系。
		StringBuffer sb1 = new StringBuffer("hello");
		StringBuffer sb2 = new StringBuffer("goodbye");
		System.out.println(sb1);
		sb1=sb2;//此时"hello"符合回收条件  
		（3） 方法内创建的对象：所创建的局部变量仅在该方法的作用期间内存在。一旦该方法返回，在这个方法内创建的对象就符合垃圾收集条件。有一种明显的例外情况，就是方法的返回对象。
		public static void main(String[] args) {
		Date d = getDate();
		System.out.println("d = " + d);
		}
		private static Date getDate() {
		Date d2 = new Date();
		StringBuffer now = new StringBuffer(d2.toString());
		System.out.println(now);
		return d2;
		}
		（4） 隔离引用：这种情况中，被回收的对象仍具有引用，这种情况称作隔离岛。若存在这两个实例，他们互相引用，并且这两个对象的所有其他引用都删除，其他任何线程无法访问这两个对象中的任意一个。也可以符合垃圾回收条件。
		public class Island {
		Island i;
		public static void main(String[] args) {
		Island i2 = new Island();
		Island i3 = new Island();
		Island i4 = new Island();
		i2.i=i3;
		i3.i=i4;
		i4.i=i2;
		i2=null;
		i3=null;
		i4=null;
		}
		}
	（问题八：垃圾收集前进行清理------finalize()方法） java提供了一种机制，使你能够在对象刚要被垃圾回收之前运行一些代码。这段代码位于名为finalize()的方法内，所有类从Object类继承这个方法。由于不能保证垃圾回收器会删除某个对象。因此放在finalize()中的代码无法保证运行。因此建议不要重写finalize();

###理解final问题有很重要的含义
	许多程序漏洞都基于此----final只能保证引用永远指向固定对象，不能保证那个对象的状态不变。
	在多线程的操作中,一个对象会被多个线程共享或修改，一个线程对对象无意识的修改可能会导致另一个使用此对象的线程崩溃。
	一个错误的解决方法就是在此对象新建的时候把它声明为final，意图使得它"永远不变"。其实那是徒劳的。 

###了解了JVM结构和垃圾回收机制后如何写出健壮代码
	1)、尽早释放无用对象的引用。 好的办法是使用临时变量的时候，让引用变量在退出活动域后，自动设置为null，暗示垃圾收集器来收集该对象，防止发生内存泄露。对于仍然有指针指向的实例，jvm就不会回收该资源,因为垃圾回收会将值为null的对象作为垃圾，提高GC回收机制效率；
	
	2)、定义字符串应该尽量使用 String str="hello"; 的形式 ，避免使用String str = new String("hello"); 的形式。因为要使用内容相同的字符串，不必每次都new一个String。例如我们要在构造器中对一个名叫s的String引用变量进行初始化，把它设置为初始值，应当这样做： 
		public class Demo {
		private String s;
		public Demo() {
		s = "Initial Value";
		}
		}
		 
		public class Demo {
		private String s;
		...
		public Demo {
		s = "Initial Value";
		}
		...
		}
		 而非
		s = new String("Initial Value");  
		s = new String("Initial Value"); 
	后者每次都会调用构造器，生成新对象，性能低下且内存开销大，并且没有意义，因为String对象不可改变，所以对于内容相同的字符串，只要一个String对象来表示就可以了。也就说，多次调用上面的构造器创建多个对象，他们的String类型属性s都指向同一个对象。    

	3)、我们的程序里不可避免大量使用字符串处理，避免使用String，应大量使用StringBuffer ，因为String被设计成不可变(immutable)类，所以它的所有对象都是不可变对象，请看下列代码；
	String s = "Hello";   
	s = s + " world!";  
	String s = "Hello";
	s = s + " world!";
	在这段代码中，s原先指向一个String对象，内容是 "Hello"，然后我们对s进行了+操作，那么s所指向的那个对象是否发生了改变呢？答案是没有。这时，s不指向原来那个对象了，而指向了另一个 String对象，内容为"Hello world!"，原来那个对象还存在于内存之中，只是s这个引用变量不再指向它了。         通过上面的说明，我们很容易导出另一个结论，如果经常对字符串进行各种各样的修改，或者说，不可预见的修改，那么使用String来代表字符串的话会引起很大的内存开销。因为 String对象建立之后不能再改变，所以对于每一个不同的字符串，都需要一个String对象来表示。这时，应该考虑使用StringBuffer类，它允许修改，而不是每个不同的字符串都要生成一个新的对象。并且，这两种类的对象转换十分容易。
	
	4)、尽量少用静态变量 ，因为静态变量是全局的，GC不会回收的；
	
	5)、尽量避免在类的构造函数里创建、初始化大量的对象 ，防止在调用其自身类的构造器时造成不必要的内存资源浪费，尤其是大对象，JVM会突然需要大量内存，这时必然会触发GC优化系统内存环境；显示的声明数组空间，而且申请数量还极大。         以下是初始化不同类型的对象需要消耗的时间：
	从表1可以看出，新建一个对象需要980个单位的时间，是本地赋值时间的980倍，是方法调用时间的166倍，而新建一个数组所花费的时间就更多了。
	
	6)、尽量在合适的场景下使用对象池技术 以提高系统性能，缩减缩减开销，但是要注意对象池的尺寸不宜过大，及时清除无效对象释放内存资源，综合考虑应用运行环境的内存资源限制，避免过高估计运行环境所提供内存资源的数量。
	
	7)、大集合对象拥有大数据量的业务对象的时候，可以考虑分块进行处理 ，然后解决一块释放一块的策略。
	
	8)、不要在经常调用的方法中创建对象 ，尤其是忌讳在循环中创建对象。可以适当的使用hashtable，vector 创建一组对象容器，然后从容器中去取那些对象，而不用每次new之后又丢弃。
	
	9)、一般都是发生在开启大型文件或跟数据库一次拿了太多的数据，造成 Out Of Memory Error 的状况，这时就大概要计算一下数据量的最大值是多少，并且设定所需最小及最大的内存空间值。
	
	10)、尽量少用finalize函数 ，因为finalize()会加大GC的工作量，而GC相当于耗费系统的计算能力。
	
	11)、不要过滥使用哈希表 ，有一定开发经验的开发人员经常会使用hash表（hash表在JDK中的一个实现就是HashMap）来缓存一些数据，从而提高系统的运行速度。比如使用HashMap缓存一些物料信息、人员信息等基础资料，这在提高系统速度的同时也加大了系统的内存占用，特别是当缓存的资料比较多的时候。其实我们可以使用操作系统中的缓存的概念来解决这个问题，也就是给被缓存的分配一个一定大小的缓存容器，按照一定的算法淘汰不需要继续缓存的对象，这样一方面会因为进行了对象缓存而提高了系统的运行效率，同时由于缓存容器不是无限制扩大，从而也减少了系统的内存占用。现在有很多开源的缓存实现项目，比如ehcache、oscache等，这些项目都实现了FIFO、MRU等常见的缓存算法

	12)、分散对象创建或删除的时间
	　　集中在短时间内大量创建新对象,特别是大对象,会导致突然需要大量内存,JVM在面临这种情况时,只能进行主GC,以回收内存或整合内存碎片,从而增加主GC的频率。集中删除对象,道理也是一样的。它使得突然出现了大量的垃圾对象,空闲空间必然减少,从而大大增加了下一次创建新对象时强制主GC的机会。

	13)、能用基本类型如Int,Long,就不用Integer,Long对象
	　　基本类型变量占用的内存资源比相应对象占用的少得多,如果没有必要,最好使用基本变量。

	14)、不要显式调用System.gc()
	　　此函数建议JVM进行主GC,虽然只是建议而非一定,但很多情况下它会触发主GC,从而增加主GC的频率,也即增加了间歇性停顿的次数。

###常用的垃圾回收算法
	1. 引用计数法(Reference Counting Collector)
	比较古老的回收算法。原理是此对象有一个引用，即增加一个计数，删除一个引用则减少一个计数。垃圾回收时，只用收集计数为0的对象。此算法最致命的是无法处理循环引用的问题。
	2. tracing算法(Tracing Collector)
	3. compacting算法(Compacting Collector)
	4. copying算法(Coping Collector)
	5. generation算法(Generational Collector)
	6. adaptive算法(Adaptive Collector)

###对象已死？
	1.引用计数（Reference Counting）:
	比较古老的回收算法。原理是此对象有一个引用，即增加一个计数，删除一个引用则减少一个计数。垃圾回收时，只用收集计数为0的对象。此算法最致命的是无法处理循环引用的问题。

	2.根搜索算法
	在主流的商用程序语言中（Java和C#，甚至包括前面提到的古老的Lisp），都是使用根搜索算法（GC Roots Tracing）判定对象是否存活的。这个算法的基本思路就是通过一系列的名为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连（用图论的话来说就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。
	
	在Java语言里，可作为GC Roots的对象包括下面几种：
		虚拟机栈（栈帧中的本地变量表）中的引用的对象。
　　　	方法区中的类静态属性引用的对象。
　　	方法区中的常量引用的对象。
　　	本地方法栈中JNI（即一般说的Native方法）的引用的对象。

###垃圾收集算法
	
	1.标记-清除（Mark-Sweep）:
	此算法执行分两阶段。第一阶段从引用根节点开始标记所有被引用的对象，第二阶段遍历整个堆，把未标记的对象清除。此算法需要暂停整个应用，同时，会产生内存碎片。

	2.复制（Copying）: 新生代 年轻代
	此算法把内存空间划为两个相等的区域，每次只使用其中一个区域。垃圾回收时，遍历当前使用区域，把正在使用中的对象复制到另外一个区域中。次算法每次只处理正在使用中的对象，因此复制成本比较小，同时复制过去以后还能进行相应的内存整理，不会出现“碎片”问题。当然，此算法的缺点也是很明显的，就是需要两倍内存空间。

	3.标记-整理（Mark-Compact）:
	此算法结合了“标记-清除”和“复制”两个算法的优点。也是分两阶段，第一阶段从根节点开始标记所有被引用对象，第二阶段遍历整个堆，把清除未标记对象并且把存活对象“压缩”到堆的其中一块，按顺序排放。此算法避免了“标记-清除”的碎片问题，同时也避免了“复制”算法的空间问题。

	4.分代收集算法
	当前商业虚拟机的垃圾收集都采用“分代收集”（Generational Collection）算法，这种算法并没有什么新的思想，只是根据对象的存活周期的不同将内存划分为几块。一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记-清理”或“标记-整理”算法来进行回收。


###垃圾收集器
	1. Serial Garbage Collector
	Serial Garbage Collector通过暂停所有应用的线程来工作。它是为单线程工作环境而设计的。它中使用一个线程来进行垃圾回收。这种暂停应用线程来进行垃圾回收的方式可能
	不太适应服务器环境。它最适合简单的命令行程序。
	
	2. Parallel Garbage Collector
	Parallel Garbage Collector也被称为吞吐量收集器（throughput collector）。它是Java虚拟机的默认垃圾收集器。与Serial Garbage Collector不同，Parallel Garbage Collector使用多个线程进行垃圾回收。与Serial Garbage Collector相似的地方时，它也是暂停所有的应用线程来进行垃圾回收。

	3. CMS Garbage Collector
	Concurrent Mark Sweep (CMS) Garbage Collector使用多个线程来扫描堆内存来标记需要回收的实例，然后再清除被标记的实例。CMS Garbage Collector只有在如下两种情景才会暂停所有的应用线程：当标记永久代内存空间中的对象时；当进行垃圾回收时，堆内存同步发生了一些变化。相比Parallel Garbage Collector，CMS Garbage Collector使用更多的CPU资源来确保应用有一个更好的吞吐量。如果分配更多的CPU资源可以获得更好的性能，那么CMS Garbage Collector是一个更好的选择，相比Parallel Garbage Collector。

	4. G1 Garbage Collector
	G1 Garbage Collector用于大的堆内存区域。它将堆内存分割成多个独立区域（Region），然后并发地对它们进行垃圾回收。在释放内存后，G1还可以压缩空闲的堆内存。但是，CMS Garbage Collector是通过“Stop The World (STW)”来进行内存压缩的。G1优先收集可回收更多内存的区域。


###经过上述的说明，可以发现垃圾回收有以下的几个特点

	（1）垃圾收集发生的不可预知性：由于实现了不同的垃圾收集算法和采用了不同的收集机制，所以它有可能是定时发生，有可能是当出现系统空闲CPU资源时发生，也有可能是和原始的垃圾收集一样，等到内存消耗出现极限时发生，这与垃圾收集器的选择和具体的设置都有关系。 

	（2）垃圾收集的精确性：主要包括2 个方面：（a）垃圾收集器能够精确标记活着的对象；（b）垃圾收集器能够精确地定位对象之间的引用关系。前者是完全地回收所有废弃对象的前提，否则就可能造成内存泄漏。而后者则是实现归并和复制等算法的必要条件。所有不可达对象都能够可靠地得到回收，所有对象都能够重新分配，允许对象的复制和对象内存的缩并，这样就有效地防止内存的支离破碎。 

	（3）现在有许多种不同的垃圾收集器，每种有其算法且其表现各异，既有当垃圾收集开始时就停止应用程序的运行，又有当垃圾收集开始时也允许应用程序的线程运行，还有在同一时间垃圾收集多线程运行。 

	（4）垃圾收集的实现和具体的JVM 以及JVM的内存模型有非常紧密的关系。不同的JVM 可能采用不同的垃圾收集，而JVM 的内存模型决定着该JVM可以采用哪些类型垃圾收集。现在，HotSpot 系列JVM中的内存系统都采用先进的面向对象的框架设计，这使得该系列JVM都可以采用最先进的垃圾收集。 

	（5）随着技术的发展，现代垃圾收集技术提供许多可选的垃圾收集器，而且在配置每种收集器的时候又可以设置不同的参数，这就使得根据不同的应用环境获得最优的应用性能成为可能。


	http://www.codeceo.com/article/jvm-memory-6-areas.html
	Heap在32位的操作系统上最大为2G，在64位的操作系统上则没有限制
































