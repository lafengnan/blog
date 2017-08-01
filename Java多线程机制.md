# Java多线程机制

## 0x1 背景

这是2016年10月阅读[JSR133 $$Java^{TM}$$内存模型和线程规范](https://jcp.org/en/jsr/detail?id=133)时留下的一篇笔记。当时笔记是记在OneNote中的，但是现在OneNote越来越反人类，想想还是整理一下搬到这里比较合适，免得哪天OneNote抽风打不开又悔之不及。

我们都知道Java虚拟机是支持多线程执行的，而线程之间的同步又是多线程的核心之一。关于Java虚拟机线程间同步的机制在*JSR133 $$Java^{TM}$$内存模型和线程规范*作了详细描述。在描述线程同步概念之前我们首先需要了解以下两个基本概念。

* 什么是线程
* 什么是线程间的同步

关于什么是线程，这个问题似乎很晦涩，因为它是一个看不见摸不着的存在，经常和“进程”搅和在一起令人不胜其烦，尤其在系统遇到问题时更是容易让人焦头烂额。按照传统的说法，“进程拥有资源，而线程只是进程的执行过程”这个解释视乎并不令人满意。[Richard Stevens](http://www.kohala.com/start/)在其大作[APUE](http://www.apuebook.com/)中写了下面一段话:

> A thread consits of the information necessary to represent an execution context within a process. This includes a thread ID that identifies the thread within a process, a set of registers, a stack, a scheduling priority and policy, a signal mask, an errno variable, and thread-specific data. Everything within a process is sharable among the threads in a process, including the text of the executable program, the programs' global and heap memory, the stacks, and the file descriptors.

大意是说，线程由表示进程内执行上下文的一些必要信息组成。这些信息包括线程ID、一组寄存器、线程栈、调度优先级与调度策略、信号掩码、errno变量、以及线程私有数据。进程中的所有资源都对其内部线程共享，包括可执行的程序代码、全局内存和堆内存、栈以及文件描述符。也就是说进程中的线程**代表了进程的执行上下文**，而进程则包括所有可访问的资源。一个单线程进程意味着整个进程的资源只给一个线程使用，不存在其他线程与其共享资源；一个多线程进程则意味着正进程中存在多个控制上下文，进程会在不同的上下文之间切换执行不同的逻辑，而这些上下文全都共享进程的全部地址空间。

既然一个进程中的所有线程都可以访问进程中的全部地址空间，那么当多个控制线程共享相同的内存时会出现什么状况？如果线程$$T_1$$可以修改某个全局变量$$V_1$$,那么其他线程$$T_i$$也可以读取或者修改$$V_1$$时就可能出现冲突，因为我们无法控制不同的线程对变量的访问顺序，这时候就需要对这些线程进行同步，以确保线程在访问变量的内容时不会访问到无效的值。

我们知道CPU执行指令是按照时钟周期进行工作的，如果程序中某条语句的执行需要超过一个时钟周期那么就可能出现一个线程对内存的读操作出现在其他线程对同一个变量的写操作中间，从而导致读出无效的数值。虽说这种无法预测的行为和CPU的体系结构有关，但是作为可移植的程序必须舍弃对CPU体系结构的依赖性假设，我们需要完全逻辑意义上的统一。下图是APUE中的一副示意图，可以用来说明线程A和线程B内存访问周期交叉时产生的数据不一致问题：

![threads_interleaves](/resources/threads_interleaves.png)

为了解决这个问题，线程就不得不使用锁来确保同一时间值允许同一个线程访问同一个变量。下面这幅图则描述了这种用锁保证的同步机制。如果线程B想读取变量，它必须首先请求变量上的锁，同样的，如果线程A需要更新该变量，它也需要首先获得锁。因此当A先拿到锁时，B就必须等待，直到A完成变量的更新操作并释放锁资源，然后再读取变量。这样，线程B读到的数据就一定是最新的有效数据，而非无效的过期数据。

![threads_interleaves_locked](/resources/threads_interleaves_locked.png)

当然，当多个线程同时对同一个变量进行写操作时也需要进行同步。事实上，只要多个线程中至少有一个写操作，那么就必须对这些线程进行同步，以确保对数据操作的结果是一致的。比如两个线程对同一个变量进行自增操作，实际上自增操作可以拆解成以下3步：

1. 从内存单元中将内容读进寄存器
2. 在寄存器中将数值加1
3. 将新的数值回写进内存单元

![threads_multi_writes](/resources/threads_multi_writes.png)

由于线程是由操作系统控制并调度的，那么线程之间的同步机制也就需要操作系统的支持。目前我们知道大部分操作系统都提供了下列几种线程同步机制：

1. 互斥量(mutex)
2. 信号量(semaphore)
3. 读写锁(reader-writer lock)
4. 条件变量(condition variable)

其中，

互斥量是一个互斥的二元锁机制，在访问共享资源前对互斥量进行加锁，在访问完成后释放互斥量上的锁，从而确保同一时间只有一个线程访问数据。对互斥量加锁以后，任何其他试图再次对互斥量加锁的线程都会被阻塞，直到当前线程释放该互斥锁。

信号量则可以看作是一个高级互斥量，互斥量是信号量只有0/1两个值时的特殊情况。信号量可以有更多的取值空间，用来实现更复杂的同步，不仅仅只是用于线程间的互斥同步上。

读写锁与互斥量类似，但是比互斥量允许更高的并行性。互斥量的状态要么是加锁的，要么是未加锁的，并且在同一时间只有一个线程可以加锁。但是读写锁却可以有**三种**状态：

* 读模式锁定
* 写模式锁定
* 未锁定

在同一时间，只有一个线程能持有一把读写锁的写模式锁，但是可以有多个线程同时持有读写锁的读模式锁，很明显这可以极大地提高读效率，使得读写锁非常适用于读操作次数远大于写操作的情景。读写锁也称为*共享-独占*锁，当读写锁以读模式锁定时，它是以共享模式锁定的；当以写模式锁定时，则是以独占模式锁定。

条件变量是另一种线程间的同步机制。条件变量与互斥量一起使用可以使线程以无竞争的方式等待特定条件的发生。条件变量本身由互斥量保护，线程在改变条件状态之前必须首先对互斥量加锁，这样其它线程在获取到互斥量之前是无法察觉条件状态的变化的，因为必须先锁定互斥量之后才能计算条件。

当然除了上述几种同步机制之外，还有很多其他的线程间同步机制，例如Linux内核2.5引入的的[RCU机制](https://www.kernel.org/doc/Documentation/RCU/whatisRCU.txt)，该机制是一种适用于多数读的应用场景的同步机制。RCU机制中，读操作无需获取锁资源就可以访问临界资源，而写操作则首先拷贝一份临界区副本进行修改，然后在所有引用该共享的临界区的引用都完成操作时将指向旧临界区的指针指向新的更新后的临界区副本。



## 0x2 Java线程同步

前面我们说的是操作系统中线程的概念以及线程间的同步，那么在Java虚拟机中线程以及线程间同步又是什么形式呢？在Java虚拟机中，线程以及被精心的做了封装，不再是pthread库里看到的一个个函数那样，而是用**Thread**类来表示了。用户创建一个线程的唯一方式就是创建一个**Thread**类的实例，然后用该对象调用start()方法启动线程。

Java中的线程同步机制也有多种，包括：

* *synchronized*关键字
* *java.util.concurrent*包中提供的各种机制
  - 可重入锁（ReentrantLock）
  - 条件变量（Condition）
  - 读写锁（ReadWriteLock）
  - 版本锁（StampedLock）
  - 各种原子类型（AtomicBoolean，AtomicInteger...)
* volatile变量

虽然Java对线程以及线程的同步机制进行了一次封装，但是实际上最终”Java线程“都要与操作系统线程关联。我没有阅读过JVM的源码，但是按照JSR133中提及的线程概念与同步机制，大致可以猜出**synchronized**关键字其实底层试用了互斥量mutex，volatile变量强迫CPU每次都从内存中存取数据而非从其自身的Cache中读取数据。其他在*java.uti.concurrent*包中提供的各种锁机制也与传统的所机制没有什么区别。

## 0x3 Java线程锁

Java中的锁是通过*monitor*来实现的，每个Java对象都与一个*monitor*关联，线程可以通过它来进行加锁和解锁操作。同一时间只有一个线程可以持有*monitor*上的锁，其他试图对该*monitor*进行加锁的操作都将被阻塞直到持有该*monitor*锁的线程释放锁资源为止。我们可以把*monitor*理解成系统级别的mutex互斥量，加锁操作和解锁操作都是对互斥量的加解锁操作，这样可以很方便将Java线程锁与传统的互斥量关联起来。

在Java程序中可以使用*synchronized*关键字对*monitor*加锁，由于*monitor*都是与某个具体的Java对象关联的，因此*synchronized*关键字的使用也就分了几种情况：

1. 修饰代码块
2. 修饰方法

对于第一种情况，*synchronized*修饰的语句需要使用一个对象的引用，然后尝试在该对象的*monitor*上执行lock操作，如果lock操作无法完成就一致等待，直到其他线程释放了该对象*monitor*上的锁。当lock操作成功之后，就执行*synchronized*修饰的代码块中的代码，一旦代码块中的代码全部执行结束（无论成功还是异常结束）都会在该对象的monitor上执行一次unlock操作释放锁资源。下面的例子是一个简单的单例模式代码，其中*instance*对象的构造通过*synchronized*关键字修饰，用来防止在多线程情境下出现构造多个实例的情况：

```Java
class Singleton {
    private static volatile Singleton instance = null;
    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (instance) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

对于第二种情况，用*synchronized*关键字修饰的方法在调用时会自动执行一次lock操作，在lock操作完成之前该方法都不会被执行。如果调用的方法是实例方法，则lock操作是作用在调用该方法的实例上的相关*monitor*上；如果是静态方法，则lock 操作作用在定义该方法的类对象的相关*monitor*上。一旦方法执行结束（无论正常还是异常结束）都会在之前执行lock操作的monitor上自动执行一次unlock操作。下面的代码简单展示了*synchronized*方法：

```java
public static synchronized Singleton getInstanceII() {
     if (instance == null) {
            instance = new Singleton();
     }
     return instance;
}
```

这两种修饰方法的区别是什么呢？从示例代码以及前文所述我们可以知道主要有两个区别：

1. 执行lock/unlock操作的monitor对象不同，*synchronized*代码块的lock/unlock操作作用在*synchronized*关键字修饰的对象所关联的monirtor上，而*synchronized*方法则作用在实例对象或者类对象上；
2. 执行lock的时机不同，*synchronized*代码块只有执行到*synchronized*修饰词时才进行lock操作而*synchronized*方法则是在方法调用时就执行lock操作。对于单例模式这种代码来说，如果用*synchronized*方法会导致明显的效率低下，因为一旦实例被构造成功，并不需要每次都进行一次lock操作继而构造对象，只要第一个条件判断不成功就可以成功绕开加锁操作，从而减少开销提高性能。



## 0x4 Java线程wait集与通知

每个Java对象除了拥有相关联的*monitor*还有一个相关联的*等待集（wait set）*，一个*wait set*是一个线程集合。当对象首次创建时，其关联的等待集是一个Empty集合，向等待集中添加线程与从等待集中删除线程的操作都是原子操作，但是只能通过下列方法操作：

* Object.wait
* Object.notify
* Object.notifyAll

等待集的操作可以被线程的中断状态以及Thread类中处理中断的方法影响。此外，Thread类中用于休眠和加入其它线程的方法都拥有从等待和通知操作继承来的属性。

### 等待（Wait）

等待动作的发生由对*wait()*方法，或者对带有时间参数的*wait(long millisecs)* 与*wait(long millisecs, int nanosecs)*方法的调用触发。用参数*0*调用*wait(long millisecs)*或者用参数*0， 0*调用*wait(long millisecs, int nanosecs)*与直接调用无参数的*wait()*方法等价。如果没有抛出*InterruptedException*异常，那么线程会从一个等待动作中正常返回。

假设线程$$t$$是对象$$m$$上执行*wait*方法的线程，设$$n$$是线程$$t$$在$$m$$上尚未执行过相应的*unlock*操作的*lock*操作的次数，那么下列操作之一会出现：

* 如果$$n = 0$$，（例如，线程$$t$$尚未在目标对象$$m$$上执行lock操作），则抛出*IllegalMonitorStateException*异常，如下面代码执行时，会在16行处抛出*java.lang.IllegalMonitorStateException*异常。因为没有对*queue*对象执行lock操作，

  ```java
  class Consumer extends Thread {
      private Queue<Integer> queue;
      private Integer maxSize;
      public Consumer(Queue<Integer> queue, int maxSize, String name) {
          super(name);
          this.queue = queue;
          this.maxSize = maxSize;
      }

      @Override
      public void run() {
          while (true){
              while (queue.isEmpty()) {
                  try {
                      System.out.println("queue is empty, consumer is waiting...");
                      queue.wait();
                  } catch (Exception e) {
                      e.printStackTrace();
                  }
              }
              int i = queue.remove();
              System.out.println("consuming value: " + i);
              queue.notifyAll();
          }
      }
  }
  ```

* 如果调用的是带时间参数的wait方法，若参数*nanosecs*不在$$0-999,999$$范围内或者*millisecs*为负数，抛出*IllegalArgumentsException*异常；

* 如果线程$$t$$被中断，则抛出*InterruptedException*异常，并且$$t$$的中断状态会被设置为$$false$$；

* 否则将会按照下列顺序出现：

  1. 线程$$t$$被添加到对象$$m$$的等待集，并在$$m$$上执行$$n$$次unlock操作
  2. 在线程$$t$$从$$m$$的等待集中移除之前，将不执行任何进一步的指令。下面的任意一个操作都可以将线程从对象的等待集中移除并在后续的某个时间恢复运行：

     - 在$$m$$上执行*notify*操作，线程$$t$$被选中从$$m$$的wait集中删除
     - 在$$m$$上执行*notifyAll*操作
     - 线程$$t$$执行一个*interrupt*中断操作
     - 如果是一个定时的*wait*操作，在超时之后（至少millisecs毫秒+nanosecs纳秒之后）内部动作就会将$$t$$从$$m$$的wait集中移除
     - JVM内部的操作也会导致线程从等待集中移除（例如，虚假唤醒—将线程从wait集移除，然后线程可以恢复执行但是又没有明确的指令让JVM做这件事，注意：这个情况需要wait只在循环中使用，而且仅在线程等待的条件满足后才推出循环）

  3. 线程$$t$$在$$m$$上执行$$n$$次lock操作
  4. 如果线程$$t$$在第二步中因为中断被从$$m$$的等待集中移除，那么$$t$$的中断状态被设置为$$false$$，并且*wait*方法会抛出*InterruptedException*


### 通知（Notification）

调用*notify*和*notifyAll*就会发生通知操作。设线程$$t$$是在对象$$m$$上执行任一上述方法的线程, $$n$$为$$t$$在$$m$$上执行的尚未执行过相应unlock 动作的 lock 动作的次数，则会发生下面的动作之一：

* 如果$$n = 0$$, 则抛出*IllegalMonitorStateException*，这是$$t$$尚未在$$m$$上执行lock操作的情况；
* 如果$$n > 0$$, 并且是一个*notify*操作，那么如果$$m$$的wait集不为空，将从$$m$$的当前wait集中选择一个成员线程$$u$$移除。但是并不存在保证说一定会选择哪一个线程。移除操作会使线程$$u$$从等待中恢复执行，但是需要注意的是$$u$$从wait状态恢复后，lock操作并不是立刻成功的，还是需要等到$$t$$对$$m$$的monitor执行了完整的unlock操作之后才可以；
* 如果$$n > 0$$,并且是一个*notifyAll*操作，那么所有的线程都会从$$m$$的等待集中移除，移除可以恢复执行。需要注意的是，这些恢复的线程中也仍然只有一个线程可以成功的对$$m$$的monitor执行lock操作



### 中断（Interruptions）

调用*Thread.interrupt*，以及会调用该方法的其他方法，比如*ThreadGroup.interrupt*都会发生中断操作。

设$$t$$为调用*u.interrupt*的线程，对于某个线程$$u$$，$$t$$和$$u$$可能是同一个线程。这个动作会导致线程$$u$$的中断状态变成$$true$$。此外，如果存在某个对象$$m$$，其wait集中包含了线程$$u$$，那么$$u$$会从$$m$$的等待集中移除，这使$$u$$可以从一个等待操作中恢复，在重新锁定$$m$$的monitor之后抛出*InterruptedException*异常。

调用*Thread.isInterrupted*可以判断一个线程的中断状态。静态方法*Thread.interrupted*可以被一个线程调用去观察并清理其自身的中断状态。



### 等待、通知与中断的交互

如果一个线程在等待时同时被通知与中断，可能会怎样？恩。。。可能会发生下面任意一种情况：

* 从*wait*中正常返回，同时仍然带有一个pending的中断（换句话说，调用*Thread.interrupted*会返回$$true$$）
* 从*wait*中抛出一个*InterruptedException*返回

线程可能并不会重置其中断状态，并从*wait*中正常返回。

同样，通知也不能因为中断而丢失。假设对象$$m$$的等待集中有一组线程$$s$$，另一个线程在$$m$$上执行了一个*notify*操作，那么会出现下面两种情况中的一种：

* 线程集合$$s$$中至少有一个线程从wait集中正常返回，或者
* 线程集合$$s$$中的所有线程必须通过抛出*InterruptedException*退出wait集

需要注意的是，如果一个线程同时处于被中断的以及被*notify*通知唤醒的状态，那么该线程需要通过抛出*InterruptedException*从wait集中返回，然后wai集中某个其他线程必须被通知到。

### Sleep和Yield

*Thread.sleep*会导致当前的执行线程根据系统定时器和调度器的精确度进入睡眠状态（暂时中断执行）并持续一定时间。线程并不丢失对任何*monitor*的所有权，线程恢复执行的情况依赖于线程的调度以及处理器是否空闲。

*Thread.yield*会导致当前的执行线程建议主动让渡出处理器供其他线程使用，但是需要注意点是：仅仅是建议！系统可以不理会该操作，因此其行为是不可捉摸的，应该尽量避免使用该方法。

需要重点强调的是，*Thread.sleep*和*Thread.yield*都没有任何同步语义。实际上，编译器并不需要在调用*Thread.sleep*和*Thread.yield*之前将寄存器中缓存的写操作回写到共享的内存，也不需要在调用*Thread.sleep*和*Thread.yield*之后重新加载缓存在寄存器中的值。在下面的例子中，假设*this.done*是 一个non-volatile布尔字段：

```java
while(!this.done)
  Thread.sleep(1000);
```

编译器可以对*this.done*只读取一次，然后在每次循环中重用缓存的值。这也意味着，即使另一个线程已经改变了*this.done*的值，循环不会终止。



## 0x5 线程状态

Java线程的状态可以从*java.lang.Thread.State*枚举类中窥其全貌，该枚举为Java线程定义了6个状态，同一时刻每个线程只能处于一种状态中，状态的详细信息如下(删除了注释信息)：

```java
public enum State {
        NEW, 
        RUNNABLE,
        BLOCKED,
        WAITING,
        TIMED_WAITING,
        TERMINATED;
}
```

其中，

* **NEW**表示一个尚未启动的线程状态；
* **RUNNALBE**表示线程可以在JVM中执行了，但也可能正在等待操作系统提供资源（比如处理器资源）；
* **BLOCKED**表示线程被monitor锁阻塞了，处于阻塞状态的线程会等待一个monitor锁以进入*synchronized*代码块或者*synchronized*方法或者在调用*Object.wait()*方法后重入*synchronized*代码块或者方法；
* **WAITING**表示线程正在等待其他线程执行一个特定的操作（比如一个执行了wait()方法的线程可能等待另一个线程调用notify()或者notifyAll()方法）。线程可以通过下述方法进入**WAITING**状态：
  - 调用*Object.wait()*方法
  - 调用*Thread类中的join()*方法
  - 调用*LockSupport.park()*方法
* **TIMED_WAITING**表示线程等待给定的时间，该状态与**WAITING**状态类似，但是需要提供超时时限。线程可以通过下属方法进入**TIMED_WAITING**状态：
  - 调用*Thread.sleep()*方法
  - 调用*Object.wait(long millisecs)*方法
  - 调用*Thread类中的join(long millisecs)*方法
  - 调用*LockSupport.parkNanos*方法
  - 调用*LockSupport.parkUntil*方法
* **TERMINATED**表示线程已经完成执行，进入终止状态

线程这几个状态的转换关系如下图所示：

![threads_state](/resources/threads_state.png)






## Appendix

[1. RCU汇总](http://www2.rdrop.com/users/paulmck/RCU/)

[2. Java语言规范（Java SE 8）](http://docs.oracle.com/javase/specs/jls/se8/html/index.html)

[3. JSR 133](https://jcp.org/en/jsr/detail?id=133)





