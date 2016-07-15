---
layout: default
title: Java 内存模型
---
<h2>{{ page.title }}</h2>
<p>{{ page.date | date_to_string }}</p>

阻碍并发的两个障碍：    
####1.指令重排
#####1.1 CPU处理器指令重排（多）
&emsp;&emsp;CPU为了提高处理速率，导致了指令重排。现在的CPU一般采用流水线来执行指令。一个指令的执行被分成：取指、译码、访存、执行、写回和更新PC等若干个阶段。然后，多条指令可以同时存在于流水线中，同时被执行。指令流水线并不是串行的，一般来说，执行占整个过程的大部分时间，并不能因为一个耗时很长的指令在“执行”阶段呆很长时间，而导致后续的指令都卡在“执行”之前的阶段上。相反，流水线是并行的，多个指令可以同时处于同一个阶段，只要CPU内部相应的处理部件未被占满即可。比如两条访存指令，可能由于第二条指令命中了cache而导致它先于第一条指令完成。一般情况下，指令乱序并不是CPU在执行指令之前刻意去调整顺序。CPU总是顺序的去内存里面取指令，然后将其顺序的放入指令流水线。但是指令执行时的各种条件，指令与指令之间的相互影响，可能导致顺序放入流水线的指令，最终乱序执行完成。这就是所谓的“顺序流入，乱序流出”。  
&emsp;&emsp;CPU的乱序执行并不是任意的乱序，而是以保证程序上下文因果关系为前提的。如a++; b＝a * 2; c--;其中a++必定先b＝a * 2执行完毕。b=a*2会等待（阻塞）a++的结果。   
#####1.2 编译器指令重排（少）
&emsp;&emsp;编译期重排序的典型就是通过调整指令顺序，在不改变程序语义的前提下，尽可能减少寄存器的读取、存储次数，充分复用寄存器的存储值。  
&emsp;&emsp;相比于CPU的乱序，编译器的乱序才是真正对指令顺序做了调整。但是编译器的乱序也必须保证程序上下文的因果关系不发生改变。  
&emsp;&emsp;如上a++; b＝a * 2; c--;因为b=a*2要等待a++的结果，编译器会拉开两条指令的时间距离，最后顺序可能是a++; c--; b＝a * 2; 
####2.高速缓存与主内存不同步
&emsp;&emsp;CPU的运算速度比主内存的读写速度要快得多，这就使得CPU在访问内存时要花很长时间来等待内存的操作，这种空等造成了系统整体性能的下降。为了解决这种速度上的不匹配问题，我们在CPU与主内存之间加入了比主内存要快的SRAM（Static Ram，静态存储器）。SRAM储存了主内存的映象，使CPU可以直接通过访问SRAM来完成数据的读写。由于SRAM的速度与CPU的速度相当，从而大大缩短了数据读写的等待时间，系统的整体速度也自然得到提高。 高速缓存即Cache，就是指介于CPU与主内存之间的高速存储器（通常由静态存储器SRAM构成）。  
&emsp;&emsp;Cache的工作原理是基于程序访问的局部性。依据局部性原理，可以在主存和CPU通用寄存器之间设置一个高速的容量相对较小的存储器，把正在执行的指令地址附近的一部分指令或数据从主存调入这个存储器，供CPU在一段时间内使用。这个介于主存和CPU之间的高速小容量存储器称作高速缓冲存储器(Cache)。  
&emsp;&emsp;CPU对存储器进行数据请求时，通常先访问Cache。由于局部性原理不能保证所请求的数据百分之百地在Cache中，这里便存在一个命中率。即CPU在任一时刻从Cache中可靠获取数据的几率。命中率越高，正确获取数据的可靠性就越大。  


    /**
    * Created by chen on 16/1/10.
    * 高速缓存与主存不同步
    * 因为CPU和内存不同步，CPU转的特别快，而内存慢，中间设计了高速缓存。CPU ＝> 高速缓存 => 内存
    * 现在多核CPU下，若A CPU跑threadX，B CPU跑threadY，Y线程修改了   isRunning的状态，
    * 但是A的高速缓存还没有被及时更新过来，X线程还是跑几毫秒
    */           
    public class MyThread {
        static boolean isRunning = true;  
        public static void threadX() {
           while (isRunning) {
            //is running...
           }
        }
        public static void threadY() {
           isRunning = false;
        }
    }  


解决办法：   
内存屏障（CPU厂商提供的方法）   
1.确保从另外一个CPU来看屏障的两边的所有都是正确的程序顺序   
2.实现内存数据可见性，确保内存数据会同步到CPU缓存子系统  
由于java找不到一个合适的设置内存屏障的方式，所以java封装了java内存模型规范，所以只要按照JMM规范，就可以做到以上两点。


####Java内存模型  
Happens-Before规则（8点）  
程序顺序规则  
传递性规则  
volatile变量规则  
监视器锁规则  
   
线程启动规则   
线程终止规则   
中断规则   

终结器规则  

前四个规则可以满足大部分需求，后三个写tomcat等容器可能用到，最后一个没见过。   


######Happens-Before
Happens-Before体现一种时序性

######程序顺序规则
在同一个线程中，出现在程序前面的动作happens-before出现在程序后面的动作
s = “hello”;System.out.println(s);//一定看到打印hello

######传递性规则
若A动作happens-before B动作，而B动作happens-before C动作，那么A动作happens-before C动作

######volatile变量规则
对volatile字段的写happens-before每一个后续的同一个字段的读
（不用考虑CPU指令重排，也不用考虑缓存的一致性）

    public class VolatileTest ｛
        static volatile String z = null;
        static void write() {
            z = “”hello, girls;
        }
        static String read() {
            return z;
        }
    ｝
    
A线程写，B线程读，可以看见立即看见z的修改。若去除volatile，则可能看到的还是z的缓存。  

######监视器锁规则  
对一个监视器的解锁happens-before每一个后续对同一个监视器的加锁。这里的监视器指的是synchronized关键字和java.uitl.concurrent.locks.Lock,他们都是具有happens-before特性的。  


import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

        /**
         * Created by chen on 16/1/10.
         * 监视器
         * A线程先执行write，B线程后执行read，可以读到最新的z值
         */
        public class MonitorLock {
            static String z = null;
            static Lock lock = new ReentrantLock();
            static void write() {
                lock.lock();
                try {
                    z = "hello world";
                } finally {
                    lock.unlock();
                }
            }
            static String read() {
                lock.lock();

                try {
                    return z;
                } finally {
                    lock.unlock();
                }
            }
        }


补充：  
volatile数组  
volatile int[] a;  
volatile数组中的每个元素是否实时可见？ 否  
解决方法？  
volatile AtomicIntegerArray a;  


final：  
final变量设置初始值后不能修改，设置可以是在定义的时候或者在构造器里面设置值。   
写final域的重排序规则禁止把final域的写重排序到构造函数之外。所以会先设值final变量，然后再返回构造器引用。
JSR-133专家组增强了final的语义。通过为final域增加写和读重排序规则，可以为java程序员提供初始化安全保证：只要对象是正确构造的（被构造对象的引用在构造函数中没有“逸出”），那么不需要使用同步（指lock和volatile的使用），就可以保证任意线程都能看到这个final域在构造函数中被初始化之后的值。  

######volatile和synchronized的区别
1.volatile仅能使用在变量级别；  
synchronized则可以使用在变量 方法 和 类级别的  
2.volatile本质是在告诉jvm当前变量在寄存器（工作内存）中的值是不确定的，需要从主存中读取；  
synchronized是锁定当前变量，只有当前线程可以访问该变量，其他线程被阻塞住  
3.volatile只能保证修改可见性，不能保证原子性  
synchronized则可以保证变量的修改可见性和原子性  
4.volatile标记的变量不会被编译器优化  
synchronized标记的变量可以被编译器优化  

相关资料：  
http://www.cs.umd.edu/~pugh/java/memoryModel/jsr133.pdf  
http://gee.cs.oswego.edu/dl/jmm/cookbook.html
    