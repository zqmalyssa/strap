---
layout: post
title: Java多线程
tags: [code, java]
author-id: zqmalyssa
---

Java多线程是java学习中比较重要也比较难懂的点，参考一些书籍总结一下

1. [Java多线程编程核心技术]

### Java中Thread类多线程基础

1. Thread类中的start()方法通知"线程规划器"此线程已经准备就绪，等待调用线程对象的run()方法，具有异步执行效果。**如果调用thread.run()就不是异步执行了，而是同步，此线程对象并不交给"线程规划期"，而是main主线程来调用run()方法**，还有，执行start()方法的顺序不代表线程启动的顺序。
2. 多线程不共享数据没事，如果多线程共享数据，就会产生"非线程安全问题"，你想的5个线程依次对一个变量做递减操作，实际打印出来可能不是递减，存在相同值
3. 在run方法前加入synchronized关键字，使多线程在执行run方法时，以排队的方式进行处理，这里要注意的是，如果print(i--)，虽然print()方法时同步的，但是i--在执行print前执行，所以有发生非线程安全概率
4. 如果是创建一个Thread对象，再将其作为参数传入Thread里，然后启动，这边就要注意了，因为用的是之后的那个实例启动的，所以跟直接启动会有一定区别 （子类如果显示调用了父类的有参构造super，则不隐式调用父类的无参构造了，个人理解）
5. 在main函数中直接调用了run方法，那run方法中的this.currentThread().getName()仍然返回的是main，而start方法就不是了
6. 线程优先级具有继承性，1-10,10优先级最高。优先级具有规则性，并不是高优先级一定在低前面，只是大概率上会比低优先级先完成。优先级具有随机性，即同上面所说
7. 守护线程在没有非守护线程后也就没有存在的必要了

### Java中多线程同步问题

1. "非线程安全"问题存在于"实例变量"中，如果是方法内部的私有变量，则不存在"非线程安全"问题
2. 多个线程共同访问1个对象中的实例变量，则有可能出现"非线程安全"问题
3. 两个线程访问同一个对象中的同步方法时一定是线程安全的
4. 多个对象多个锁，3中那个线程先执行带synchronized的方法，哪个线程就持有**该方法所属对象的锁lock**，其他线程只能等待，但创建了多个对象，就不是这样了，异步执行起来了
5. A线程先持有object对象的lock锁，B线程仍可以以异步的方式调用object对象的非synchronized类型的方法，但是，B线程如果在这时调用object对象中的synchronized类型的方法则需要等待，**也就是同步**
6. 脏读也是没加synchronized引起的，因为读的时候可以随时读取未加synchronized关键字的方法，就会出现数据的不一致
7. "可重入锁"，一个线程获得了某个对象的锁，此时这个对象还没有释放，当其再次想要获取这个对象锁的时候还是可以获取的（就是类中的其他synchronized方法），"可重入锁"也支持父子类继承的环境，子类继承父类的synchronized方法，但是！！如果继承的某个方法，前面不加synchronized，是不同步的，即重写的方法仍需要加synchronized
8. A线程出现异常，锁自动释放
9. synchronized作用在方法上某些情况是有弊端的，A线程调用同步方法执行了很长时间，B线程就必选等待很长时间
10. 使用同步代码块解决这个问题synchronized(this){}，不在synchronized块中就是异步执行，在synchronized块中就是同步执行
11. 当一个线程访问object的一个synchronized(this){}同步代码块，其他线程对**同一个object**中所有其他synchronized(this){}同步代码块的访问将被阻塞，这说明synchronized使用的"对象监视器"是一个。这个要注意，跟同步一个方法，不能访问其他同步方法是一个意思。这两种一样的，synchronized(this){}，那么其他synchronized(this){}和synchronized都将被阻塞。
12. 和synchronized一样，synchronized(this){}是锁定当前对象的，即其他非synchronized方法仍能执行
13. 支持"任意对象"作为"对象监听器"来实现同步的功能，synchronized(非this对象){}，在用synchronized(非this对象){}时，对象监视器必须是同一个对象，如果不是同一个对象监视器，运行的结果就是异步调用了，**参数对象在项目中是一份实例，是单例的**。
	- 这个任意对象大多数是实例变量及方法的参数
	- 使用"Synchronized(非this对象x)同步代码块"格式进行同步操作，对象监视器必须是同一个对象，如果不是同一个对象监视器，运行的结果就是异步调用了
	- "Synchronized(非this对象x)同步代码块"具有一定的优点，如果一个类中有很多个synchronized方法，这时虽然能实现同步，但会受到阻塞，所以影响运行效率，但如果使用"Synchronized(非this对象x)同步代码块"，则代码块中的程序与同步方法是异步的的，不与其他锁this同步方法争抢this锁，可以大大提高运行效率
14. a.当多个线程同时执行synchronized(x){}同步代码块时呈同步效果，如果其中x不是一个对象也不行，执行结果是异步的 b.当其他线程执行x对象中的synchronized同步方法时呈同步效果 c.当其他线程执行x对象方法里面的synchronized(this)代码块时也呈现同步效果
15. synchronized关键字加到static静态方法上是给Class类上锁，而加到非static方法上是给对象上锁
16. 有static方法和非static方法，都用了synchronized关键字，会导致异步，异步的原因是持有了不同的锁，一个是对象锁，一个是class锁，而class锁可以对类的所有对象实例起作用，去掉非static部分的方法，恢复了同步。这边很奇怪！！synchronized(class)的代码块和synchronized static方法作用是一样的
17. String有常量池的概念，也就是a，b两个不同的String实例，值一样，对象也一样，所以如果synchronized(string)的时候要特别注意，所以同步代码块都不使用String作为锁对象，改用其他的，比如new Object()实例化一个Object对象，但它并不放入缓存中
18. 死锁的检测可以使用bin目录下的jps命令，查看run的id值是3244，再执行jstack命令查看`jstack -l 3244`
19. 将任何数据类型作为同步锁时，需要注意的是，是否有多个线程同时持有锁对象，如果同时持有相同的锁对象，则这些线程之间就是同步的，如果分别获得锁对象，线程之间就是异步的。还有，只要对象不变，即使对象的属性改变，运行的结果还是同步的，比如set一个变量
20. volatile关键字，当线程访问实例变量时，强制从公共堆栈中进行取值，但是volatile最致命的缺点是不支持原子性
21. synchronized和volatile的比较，volatile是线程同步的轻量级实现，性能肯定比synchronized好，并且volatile只能修饰于变量，而synchronized可以修饰方法以及代码块，多线程访问volatile不会阻塞，而synchronized会阻塞，volatile解决的其实是变量在多个线程之间的可见性，不具备同步性，synchronized块带有volatile的效果

### Java中线程间通信

1. wait()方法的作用是使当前执行代码的线程进行等待，在调用wait()方法前，线程必须获得该对象的对象级别锁，即只有在同步方法和同步块中调用wait()方法。notify()方法也是要在同步方法和同步块中调用。所以调用前，线程也必须获得该对象的对象级别锁。
2. notify()执行后并不立即释放锁，只是进入**就绪**状态，必须执行完notify()方法所在的同步synchronized代码块后才释放锁
3. wait(long)方法可以等待指定时间，然后自我唤醒
4. 生产消费者模式是wait()和notify()的实现，1个生产者1个消费者，1个生产者多个消费者，多个生产者1个消费者，多个生产者多个消费者
5. 通过管道进行线程间通信？？
6. 方法join()的作用是等待线程对象销毁，主线程希望等子线程执行完后，取某个数据执行，用join()
	- 来看个join()的简单例子
	```java
	package com.qiming.join;

public class MyThread extends Thread {

  @Override
  public void run() {
    try {
      int secondValue = (int)(Math.random() * 10000);
      System.out.println(secondValue);
      Thread.sleep(secondValue);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }


}
	```
	运行一下测试类就能看到效果
	```java
	package com.qiming.join;

/**
 * 测试join的作用
 * join的作用是使所属的线程对象x正常执行run()方法中的任务，而使当前线程z进行无限期的阻塞，等待线程x销毁后再执行线程z后面的代码
 * 方法join具有使线程排队运行的作用，有些类似同步的运行效果，join和synchronized的区别是join在内部使用wait()方法，而synchronized关键字使用的是"对象监视器"
 */
public class TestJoin {

  public static void main(String[] args) {
    try {
      MyThread threadTest = new MyThread();
      threadTest.start();
      threadTest.join();
      System.out.println("我想当threadTest对象执行完毕后我再执行，我做到了");
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

}

	```
7. join(long)内部调用wait()，而sleep()不是，所以wait()释放对象锁，而sleep()不释放
	- 来
8. ThreadLocal每个线程都有自己的共享变量，数据是隔离的，第一次调用ThreadLocal类的get()方法返回值是null，覆盖initialValue()方法具有初始值
9. InheritableThreadLocal类可以让子线程取得父线程继承下来的值

### Java中Lock的使用

1. ReentrantLock类，java1.5引进，也可以实现线程之间的同步互斥，lock.lock()方法获取锁，lock.unlock()方法释放锁，
2. 配合condition使用，lock中的lock和unlock就相当于synchronized，而condition的await和signal就相当于wait和notify，可以用lock去new出多个condition进行分类，然后指定condition进行signal，其他的condition就不会触发signal
	- **在用condition.await()之前调用lock.lock()代码获得同步监视器！！await()方法使当前执行任务的线程进入到了WAITING状态！！**
	- Object中的wait和notify方法相当于condition中的await和signal
	- 使用ReentrantLock对象跟condition配合可以唤醒指定种类的线程，这是控制部分线程行为的方便方式
3. 锁Lock分为公平锁和非公平锁，公平锁按照线程加锁的顺序来，FIFO，非公平锁就是随机的，Lock中有很多配置的方法，了解一下
4. ReentrantReadWriteLock类，多个Thread可以同时进行读取操作，但是同一时刻只允许一个Thread进行写入操作，lock后有readLock和writeLock两种锁，读读共享，写写互斥，读写互斥
	- 因为ReentrantLock具有完全互斥排他效果，即同一时间只有一个线程在执行ReentrantLock.lock()方法后面的任务。这样做虽然保证了实例变量的线程安全性，但效率却是非常低下的，所以JDK中提供了一种读写锁ReentrantReadWriteLock，使它可以加快运行效率，在某些不需要操作实例变量的方法中，完全可以使用读写锁ReentrantReadWriteLock来提升该方法的代码速度。
	- 读写锁表示也有两个锁，一个是读操作相关的锁，也称为共享锁，另一个是写操作相关的锁，也叫排他锁

### CountDownLatch和CyclicBarrier的使用与区别

CountDownLatch和CyclicBarrier都有让多个线程等待同步然后再开始下一步动作的意思，但是CountDownLatch的下一步的动作实施者是主线程，具有不可重复性；而CyclicBarrier的下一步动作实施者还是“其他线程”本身，具有往复多次实施动作的特点
	- 类CountDownLatch所提供的功能是判断count计数不为0时则当前线程呈wait状态，也就是在屏障处等待
	- 类CyclicBarrier不仅有CountDownLatch锁具有的功能，还可以实现屏障等待的功能，也就是阶段性同步，它在使用上的意义在于可以循环地实现线程要一起做任务的目标，而不是像类CountDownLatch一样，仅仅支持一次线程与同步点阻塞的特性，CyclicBarrier类的公共屏障点可以重用

举相应的例子

```java
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class CountDownLatchTest {

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(4);
        for(int i = 0; i < latch.getCount(); i++){
            new Thread(new MyThread(latch), "player"+i).start();
        }
        System.out.println("正在等待所有玩家准备好");
        latch.await();
        System.out.println("开始游戏");
    }

    private static class MyThread implements Runnable{
        private CountDownLatch latch ;

        public MyThread(CountDownLatch latch){
            this.latch = latch;
        }

        @Override
        public void run() {
            try {
                Random rand = new Random();
                int randomNum = rand.nextInt((3000 - 1000) + 1) + 1000;//产生1000到3000之间的随机整数
                Thread.sleep(randomNum);
                System.out.println(Thread.currentThread().getName()+" 已经准备好了, 所使用的时间为 "+((double)randomNum/1000)+"s");
                latch.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
}
```
对于CyclicBarrier，假设有一家公司要全体员工进行团建活动，活动内容为翻越三个障碍物，每一个人翻越障碍物所用的时间是不一样的。但是公司要求所有人在翻越当前障碍物之后再开始翻越下一个障碍物，也就是所有人翻越第一个障碍物之后，才开始翻越第二个，以此类推。类比地，每一个员工都是一个“其他线程”。当所有人都翻越的所有的障碍物之后，程序才结束。而主线程可能早就结束了，这里我们不用管主线程。

我们使用代码来模拟上面的过程。我们设置了三个员工和三个障碍物。可以看到所有的员工翻越了第一个障碍物之后才开始翻越第二个的

```java
package com.huai.thread;

import java.util.Random;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierTest {
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(3);
        for(int i = 0; i < barrier.getParties(); i++){
            new Thread(new MyRunnable(barrier), "队友"+i).start();
        }
        System.out.println("main function is finished.");
    }


    private static class MyRunnable implements Runnable{
        private CyclicBarrier barrier;

        public MyRunnable(CyclicBarrier barrier){
            this.barrier = barrier;
        }

        @Override
        public void run() {
            for(int i = 0; i < 3; i++) {
                try {
                    Random rand = new Random();
                    int randomNum = rand.nextInt((3000 - 1000) + 1) + 1000;//产生1000到3000之间的随机整数
                    Thread.sleep(randomNum);
                    System.out.println(Thread.currentThread().getName() + ", 通过了第"+i+"个障碍物, 使用了 "+((double)randomNum/1000)+"s");
                    this.barrier.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}

```

### Semaphore的使用

Semaphore是一种基于计数的信号量。它可以设定一个阈值，基于此，多个线程竞争获取许可信号，做自己的申请后归还，超过阈值后，线程申请许可信号将会被阻塞。Semaphore可以用来构建一些对象池，资源池之类的，比如数据库连接池，我们也可以创建计数为1的Semaphore，将其作为一种类似互斥锁的机制，这也叫二元信号量，表示两种互斥状态。它的用法如下：
availablePermits函数用来获取当前可用的资源数量
wc.acquire(); //申请资源
wc.release();// 释放资源

给个例子，餐厅2个座位，但是有3个人要等位就餐

### Executor框架

Executor框架主要由3大部分组成：

1、任务： 包括被执行的任务需要实现的接口：Runable 接口、Callable接口；
2、任务的执行： 包括任务执行机制的核心接口Executor，以及继承自Executor的ExecutorService接口。Executor框架有两个关键类实现了ExecutorService接口：ThreadPoolExecutor 和 ScheduledThreadPoolExecutor、ForkJoinPool；
3、任务的异步计算结果： 包括Future接口和实现Future接口的FutureTask类、ForkJoinTask类。

![ExecutorFrame]({{ "/assets/img/java/ExecutorFrame.png" | relative_url}})

1、Runnable接口 和 Callable接口
Runnable接口和Callable接口的实现类，都可以被ThreadPoolExecutor或Scheduled-ThreadPoolExecutor执行。它们之间的区别是Runnable不会返回结果，而Callable可以返回结
果。除了可以自己创建实现Callable接口的对象外，还可以使用工厂类Executors来把一个Runnable包装成一个Callable。

```java
//Executors方法
public static Callable<Object> callable(Runnable task);
public static <T> Callable<T> callable(Runnable task, T result);
```
2、Executor、ExecutorService、AbstractExecutorService、ScheduledExecutorService
Executor 接口： 是Executor框架的基础，它将任务的提交与任务的执行分离开来。
ExecutorService 接口： 扩展了Executor接口，提供了管理终止的方法（shutdown() ,etc），以及可为跟踪一个或多个异步任务执行状况而生成 Future 的方法。
AbstractExecutorService 类： 提供 ExecutorService 执行方法的默认实现。
ScheduledExecutorService 接口： 一个特殊的 ExecutorService，提供了 可安排在给定的延迟后运行或定期执行的命令。

3、ThreadPoolExecutor
ThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建3种类型的ThreadPoolExecutor：SingleThreadExecutor、FixedThreadPool和CachedThreadPool。
以下是这三种线程池的应用场景说明：
	- FixedThreadPool适用于为了满足资源管理的需求，而需要限制当前线程数量的应用场景，它适用于负载比较重的服务器。
	- SingleThreadExecutor适用于需要保证顺序地执行各个任务；并且在任意时间点，不会有多个线程是活动的应用场景。
	- CachedThreadPool是大小无界的线程池，适用于执行很多的短期异步任务的小程序，或者是负载较轻的服务器

4、ScheduledThreadPoolExecutor
ScheduledThreadPoolExecutor通常使用工厂类Executors来创建。Executors可以创建2种类型的ScheduledThreadPoolExecutor，如下。
ScheduledThreadPoolExecutor：包含若干个线程的ScheduledThreadPoolExecutor。
SingleThreadScheduledExecutor：只包含一个线程的ScheduledThreadPoolExecutor

5、Future 接口
Future接口和实现Future接口的FutureTask类用来表示异步计算的结果。当我们把Runnable接口或Callable接口的实现类提交（submit）给ThreadPoolExecutor或ScheduledThreadPoolExecutor时，ThreadPoolExecutor或ScheduledThreadPoolExecutor会向我们返回一个FutureTask对象。下面是对应的API。
```java
<T> Future<T> submit(Callable<T> task)
<T> Future<T> submit(Runnable task, T result)
Future<> submit(Runnable task)
```

### Java中Timer定时器

定时计划任务Timer在内部使用的是多线程的方式进行处理，所以它和线程技术还是有非常大的关联的

1. 创建一个Timer就是启动一个新的线程，这个新启动的线程并不是守护线程，可以改成创建成守护线程，这样主程序跑完也就结束了
2. timer和timeTask有不少方法，了解一下

### Java中单例模式与多线程结合

1. 知道什么是饿汉模式，什么是懒汉模式，使用DCL双检查锁机制来实现多线程环境中的延迟加载单例设计模式，使用静态内置类也可以解决

单例到底有几种实现，饿汉模式，是在调用方法前，实例已经被创建了，相对的就是懒汉模式，在调用方法时，实例才被创建，但要小心多线程情况下的问题

2. 除了懒汉恶汉，DCL解决多线程，还有什么方式实现呢

使用静态内置类实现单例模式（线程安全，但是遇到序列化对象时，得到的结果还是多例的）

序列化和反序列化（调用readResolve解决）

使用static代码块（线程安全）

enum类型实现单例模式（线程安全，在使用枚举类型时，构造方法会被自动调用）

### 查漏补缺

这边会介绍线程的状态及怎么切换的，还是很实用的，方便定位问题

### ThreadPoolExecutor

ThreadPoolExecutor是并发包中的一个线程池服务，可以很容易的将一个实现了Runbable接口的任务放入线程池中执行，想要较好的使用ThreadPoolExecutor，就要配置好corePoolSize，最大线程数，任务缓冲的队列，线程池满的处理策略，即构造函数的参数

1. 高性能，选用SynchronousQueue
2. 缓冲执行，希望Runnable任务尽量被corePoolSize范围的线程执行掉，可使用ArrayBlockingQueue或LinkedBlockingQueue作为任务的缓冲队列，这样，当线程数等于或超过corePoolSize后，会先加入到缓冲队列，而不是直接交由线程执行，这样就支持了最大线程数+BlockingQueue大小的任务数

有些方便创建ThreadPoolExecutor的方法

1. newFixedThreadPool(int)
2. newSingleThreadExecutor()
3. newCachedThreadPool()
4. newSchedulerThreadPool()
	- 分布式java中，在异步操作时需要超时回调的场景，这种情况下SchedulerThreadPoolExecutor是不错的选择，JDK5之前一直是借助Timer来实现，**Timer与其的区别在于，Timer只能单线程，一旦task执行缓慢，就会导致其他task的执行推迟，使用scheduler则可以自行控制线程数，Timer中的task会抛出RuntimeException，会导致Timer中的所有task都不能执行，Scheduler可执行Callable的task，从而在执行完毕后得到执行结果**

而线程池不允许使用Executors去创建，而要通过ThreadPoolExecutor方式，这一方面是由于jdk中Executor框架虽然提供了如newFixedThreadPool()、newSingleThreadExecutor()、newCachedThreadPool()等创建线程池的方法，但都有其局限性，不够灵活；另外由于前面几种方法内部也是通过ThreadPoolExecutor方式实现，使用ThreadPoolExecutor有助于大家明确线程池的运行规则，创建符合自己的业务场景需要的线程池，避免资源耗尽的风险

构造函数的参数含义如下：

1、corePoolSize:指定了线程池中的线程数量，它的数量决定了添加的任务是开辟新的线程去执行，还是放到workQueue任务队列中去；
2、maximumPoolSize:指定了线程池中的最大线程数量，这个参数会根据你使用的workQueue任务队列的类型，决定线程池会开辟的最大线程数量；
3、keepAliveTime:当线程池中空闲线程数量超过corePoolSize时，多余的线程会在多长时间内被销毁；
4、unit:keepAliveTime的单位
5、workQueue:任务队列，被添加到线程池中，但尚未被执行的任务；它一般分为直接提交队列、有界任务队列、无界任务队列、优先任务队列几种；
6、threadFactory:线程工厂，用于创建线程，一般用默认即可；
7、handler:拒绝策略；当任务太多来不及处理时，如何拒绝任务；

workQueue任务队列：

1. 直接提交队列（SynchronousQueue）
	- SynchronousQueue是一个特殊的BlockingQueue（阻塞队列），它没有容量，每执行一个插入操作就会阻塞，需要再执行一个删除操作才会被唤醒，反之每一个删除操作也都要等待对应的插入操作
	- 可以看到，当任务队列为SynchronousQueue，创建的线程数大于maximumPoolSize时，直接执行了拒绝策略抛出异常
	- 使用SynchronousQueue队列，提交的任务不会被保存，总是会马上提交执行。如果用于执行任务的线程数量小于maximumPoolSize，则尝试创建新的进程，如果达到maximumPoolSize设置的最大值，则根据你设置的handler执行拒绝策略。因此这种方式你提交的任务不会被缓存起来，而是会被马上执行，在这种情况下，你需要对你程序的并发量有个准确的评估，才能设置合适的maximumPoolSize数量，否则很容易就会执行拒绝策略
	- 其实就是没有容量的队列，需要立马有线程去接手完成任务，如果线程数到了maximumPoolSize，不能再接了，就直接报错了
2. 有界的任务队列（ArrayBlockingQueue）
	- 使用ArrayBlockingQueue有界任务队列，若有新的任务需要执行时，线程池会创建新的线程，直到创建的线程数量达到corePoolSize时，则会将新的任务加入到等待队列中。**若等待队列已满，即超过ArrayBlockingQueue初始化的容量**，则继续创建线程，直到线程数量达到maximumPoolSize设置的最大线程数量，若大于maximumPoolSize，则执行拒绝策略。在这种情况下，线程数量的上限与有界任务队列的状态有直接关系，如果有界队列初始容量较大或者没有达到超负荷的状态，线程数将一直维持在corePoolSize以下，反之当任务队列已满时，则会以maximumPoolSize为最大线程数上限
	- 为什么是有界，就是到corePoolSize的时候不立马去再创建大于corePoolSize的线程，还是先放到队列里面缓一缓，队列里面都满了，缓不住了，再创建大于corePoolSize的线程直到maximumPoolSize
3. 无界的任务队列（LinkedBlockingQueue）
	- 使用无界任务队列，线程池的任务队列可以无限制的添加新的任务，而线程池创建的最大线程数量就是你corePoolSize设置的数量，**也就是说在这种情况下maximumPoolSize这个参数是无效的**，哪怕你的任务队列中缓存了很多未执行的任务，当线程池的线程数达到corePoolSize后，就不会再增加了；若后续有新的任务加入，则直接进入队列等待，当使用这种任务队列模式时，一定要注意你任务提交与处理之间的协调与控制，不然会出现队列中的任务由于无法及时处理导致一直增长，直到最后资源耗尽的问题。
	- 就是队列无限大，也就是缓冲无限大，那么maximumPoolSize其实没什么用，线程一直维持在最多corePoolSize，多了你就排队等呗
4. 优先任务队列（PriorityBlockingQueue）
	- PriorityBlockingQueue它其实是一个特殊的无界队列，它其中无论添加了多少个任务，线程池创建的线程数也不会超过corePoolSize的数量，只不过其他队列一般是按照先进先出的规则处理任务，而PriorityBlockingQueue队列可以自定义规则根据任务的优先级顺序先后执行

任务拒绝策略

这里先假设一个前提：线程池有一个任务队列，用于缓存所有待处理的任务，正在处理的任务将从任务队列中移除。因此在任务队列长度有限的情况下就会出现新任务的拒绝处理问题，需要有一种策略来处理应该加入任务队列却因为队列已满无法加入的情况。另外在线程池关闭的时候也需要对任务加入队列操作进行额外的协调处理

RejectedExecutionHandler提供了四种方式来处理任务拒绝策略

1. 直接丢弃（DiscardPolicy）
2. 丢弃队列中最老的任务(DiscardOldestPolicy)
3. 抛异常(AbortPolicy)
4. 将任务分给调用线程来执行(CallerRunsPolicy)

简单解释如下，这四种策略是独立无关的，是对任务拒绝处理的四中表现形式。最简单的方式就是直接丢弃任务。但是却有两种方式，到底是该丢弃哪一个任务，比如可以丢弃当前将要加入队列的任务本身（DiscardPolicy）或者丢弃任务队列中最旧任务（DiscardOldestPolicy）。丢弃最旧任务也不是简单的丢弃最旧的任务，而是有一些额外的处理。除了丢弃任务还可以直接抛出一个异常（RejectedExecutionException），这是比较简单的方式。抛出异常的方式（AbortPolicy）尽管实现方式比较简单，但是由于抛出一个RuntimeException，因此会中断调用者的处理过程。除了抛出异常以外还可以不进入线程池执行，在这种方式（CallerRunsPolicy）中任务将由调用者线程去执行

此外，如果我们使用的是springboot进行开发，那么可以尝试下springboot提供的@Async注解，它用到了线程中`ThreadPoolTaskExecutor`
这边可能说的有点不对 默认是`SimpleAsyncTaskExecutor`，可以看下Spring的源码

而Spring的`ThreadPoolTaskExecutor`，和原生的有什么不同，个人的理解，使用上的

1、首先拒绝策略和上面是一样的，有4种
2、队列，个人认为就是有界队列，好像没有设置队列的地方，默认值是2147483647，也就是int型的最大max值
3、用队列的话，估好值，不用队列的话，默认的很大，那么max的效果不一定能出来


这里还需要将的就是线程池的启动机制了，一定是结合上面的队列，拒绝策略进行的，下面是重点：

1、创建一个线程池，在还没有任务提交的时候，默认线程池里面是没有线程的。当然，你也可以调用prestartCoreThread方法，来预先创建一个核心线程。
2、线程池里还没有线程或者线程池里存活的线程数小于核心线程数corePoolSize时，这时对于一个新提交的任务，线程池会创建一个线程去处理提交的任务。当线程池里面存活的线程数小于等于核心线程数corePoolSize时，线程池里面的线程会一直存活着，就算空闲时间超过了keepAliveTime，线程也不会被销毁，而是一直阻塞在那里一直等待任务队列的任务来执行。
3、当线程池里面存活的线程数已经等于corePoolSize了，这是对于一个新提交的任务，会被放进任务队列workQueue排队等待执行。而之前创建的线程并不会被销毁，而是不断的去拿阻塞队列里面的任务，当任务队列为空时，线程会阻塞，直到有任务被放进任务队列，线程拿到任务后继续执行，执行完了过后会继续去拿任务。这也是为什么线程池队列要是用阻塞队列。
4、当线程池里面存活的线程数已经等于corePoolSize了,并且任务队列也满了，这里假设maximumPoolSize>corePoolSize(如果等于的话，就直接拒绝了),这时如果再来新的任务，线程池就会继续创建新的线程来处理新的任务，知道线程数达到maximumPoolSize，就不会再创建了。这些新创建的线程执行完了当前任务过后，在任务队列里面还有任务的时候也不会销毁，而是去任务队列拿任务出来执行。在当前线程数大于corePoolSize过后，线程执行完当前任务，会有一个判断当前线程是否需要销毁的逻辑：如果能从任务队列中拿到任务，那么继续执行，如果拿任务时阻塞（说明队列中没有任务），那超过keepAliveTime时间就直接返回null并且销毁当前线程，直到线程池里面的线程数等于corePoolSize之后才不会进行线程销毁。
5、如果当前的线程数达到了maximumPoolSize，并且任务队列也满了，这种情况下还有新的任务过来，那就直接采用拒绝的处理器进行处理。默认的处理器逻辑是抛出一个RejectedExecutionException异常。你也就可以指定其他的处理器，或者自定义一个拒绝处理器来实现拒绝逻辑的处理（比如讲这些任务存储起来）。JDK提供了四种拒绝策略处理类：AbortPolicy（抛出一个异常，默认的），DiscardPolicy（直接丢弃任务），DiscardOldestPolicy（丢弃队列里最老的任务，将当前这个任务继续提交给线程池），CallerRunsPolicy（交给线程池调用所在的线程进行处理）。




还有补充下corePoolSize的设置，先运行如何代码查看设置

```java
package com.qiming;

public class TestAvailableProcessors {

  public static void main(String[] args) {
    System.out.println(Runtime.getRuntime().availableProcessors());
  }

}
```
1.cpu密集型：
CPU密集的意思是该任务需要大量的运算，而没有阻塞，CPU一直全速运行。
CPU密集任务只有在真正的多核CPU才可能得到加速（通过多线程）。
/而在单核CPU上，无论你开几个模拟的多线程该任务都不可能得到加速，因为CPU总的运算能力就那些。（不过现在应该没有单核的CPU了吧）/
CPU密集型的任务配置尽可能少的线程数量：
一般公式：CPU核数+1个线程的线程池。

2.IO密集型：（分两种）：
1.由于IO密集型任务的线程并不是一直在执行任务，则应配置尽可能多的线程，如CPU核数*2
2.IO密集型，即任务需要大量的IO，即大量的阻塞。在单线程上运行IO密集型的任务会导致浪费大量的CPU运算能力浪费在等待。所以在IO密集型任务中使用多线程可以大大的加速程序运行。故需要·多配置线程数：
参考公式：CPU核数/（1-阻塞系数 ） 阻塞系数在（0.8-0.9）之间
比如8核CPU：8/（1-0.9） = 80个线程数

![corenum]({{ "/assets/img/javathread/corenum.png" | relative_url}})

上面例子本机输出是12，这边对应的也是12


这边也需要讲一些实际使用的例子：

比如ScheduledThreadPoolExecutor，注意下面的注释

```java

package com.qiming.testkafka.schedulePool;

import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

/**
 * Created by qmzhang on 2022/03/03
 */
public class MyThread implements Runnable {

    private int num;

    public MyThread(int num) {
        this.num = num;
    }

    @Override public void run() {
        int sleep = RandomNum.randomNumbers(2,5, 1).get(0);
        try {
            TimeUnit.SECONDS.sleep(sleep);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + "进来了." + "跑的任务是 " + num + " 执行时间是 " + sleep + " 当前时间是 " + LocalDateTime.now());
    }
}


package com.qiming.testkafka.schedulePool;

import java.time.LocalDateTime;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * Created by qmzhang on 2022/03/03
 */
public class TestScheduledExecutor {


    public static void main(String[] args) throws Exception{


        // 两种方法 注意
        ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(1);

        ScheduledThreadPoolExecutor scheduledThreadPoolExecutor = new ScheduledThreadPoolExecutor(5);

        System.out.println("Current Time = "+ LocalDateTime.now());

//        Thread thread = new Thread(new Runnable() {
//            @Override public void run() {
//                System.out.println(Thread.currentThread().getName() + "进来了." + " 当前时间是 " + LocalDateTime.now());
//            }
//        });

        // 初始化了5个线程, 每次10个，那么每个线程均摊2个，下次再跑的时候又重新分配每个任务的各个线程了
        // 初始化了1个线程, 每次10个，那么只有一个线程跑10个，delay基本不等待，是同时执行的，1个线程负责提交10个任务？？1000个也差不多

        // 前面两个其实是太快了，不是提交

        // 在run里加点停顿逻辑（2秒，4秒都一样，如果是间隔3秒的话）

        // 就变成了同时跑的只有5个（线程数），后面接着跑，但是每个任务的每次线程都不一样的

        // 所以线程是回收再分配的，不是一个任务一直占着直到结束

        /**
        pool-2-thread-1进来了.跑的任务是 0 当前时间是 2022-03-03T21:33:12.191
        pool-2-thread-3进来了.跑的任务是 2 当前时间是 2022-03-03T21:33:12.192
        pool-2-thread-2进来了.跑的任务是 1 当前时间是 2022-03-03T21:33:12.192
        pool-2-thread-5进来了.跑的任务是 4 当前时间是 2022-03-03T21:33:12.192
        pool-2-thread-4进来了.跑的任务是 3 当前时间是 2022-03-03T21:33:12.192
        pool-2-thread-1进来了.跑的任务是 5 当前时间是 2022-03-03T21:33:16.192
        pool-2-thread-2进来了.跑的任务是 7 当前时间是 2022-03-03T21:33:16.193
        pool-2-thread-4进来了.跑的任务是 9 当前时间是 2022-03-03T21:33:16.193
        pool-2-thread-5进来了.跑的任务是 8 当前时间是 2022-03-03T21:33:16.193
        pool-2-thread-3进来了.跑的任务是 6 当前时间是 2022-03-03T21:33:16.193
        pool-2-thread-1进来了.跑的任务是 0 当前时间是 2022-03-03T21:33:20.193
        pool-2-thread-4进来了.跑的任务是 2 当前时间是 2022-03-03T21:33:20.194
        pool-2-thread-5进来了.跑的任务是 3 当前时间是 2022-03-03T21:33:20.194
        pool-2-thread-2进来了.跑的任务是 1 当前时间是 2022-03-03T21:33:20.194
        pool-2-thread-3进来了.跑的任务是 4 当前时间是 2022-03-03T21:33:20.194
        pool-2-thread-1进来了.跑的任务是 5 当前时间是 2022-03-03T21:33:24.193
        pool-2-thread-4进来了.跑的任务是 6 当前时间是 2022-03-03T21:33:24.194
        pool-2-thread-2进来了.跑的任务是 8 当前时间是 2022-03-03T21:33:24.194
        pool-2-thread-5进来了.跑的任务是 7 当前时间是 2022-03-03T21:33:24.194
        pool-2-thread-3进来了.跑的任务是 9 当前时间是 2022-03-03T21:33:24.194
         */

        // 加了点随机值的话 就不一样了啊，应该也有抢占的概念，有的任务就会多执行一次，应该是任务都是想3秒跑一次，3秒的时候有没有线程另说
        /**
         pool-2-thread-2进来了.跑的任务是 1 执行时间是 2 当前时间是 2022-03-03T21:45:13.336
         pool-2-thread-5进来了.跑的任务是 4 执行时间是 3 当前时间是 2022-03-03T21:45:14.336
         pool-2-thread-3进来了.跑的任务是 2 执行时间是 3 当前时间是 2022-03-03T21:45:14.336
         pool-2-thread-1进来了.跑的任务是 0 执行时间是 4 当前时间是 2022-03-03T21:45:15.336
         pool-2-thread-4进来了.跑的任务是 3 执行时间是 4 当前时间是 2022-03-03T21:45:15.336
         pool-2-thread-2进来了.跑的任务是 5 执行时间是 3 当前时间是 2022-03-03T21:45:16.336
         pool-2-thread-5进来了.跑的任务是 6 执行时间是 2 当前时间是 2022-03-03T21:45:16.336
         pool-2-thread-3进来了.跑的任务是 7 执行时间是 3 当前时间是 2022-03-03T21:45:17.337
         pool-2-thread-4进来了.跑的任务是 9 执行时间是 3 当前时间是 2022-03-03T21:45:18.336
         pool-2-thread-1进来了.跑的任务是 8 执行时间是 4 当前时间是 2022-03-03T21:45:19.336
         pool-2-thread-5进来了.跑的任务是 1 执行时间是 3 当前时间是 2022-03-03T21:45:19.337
         pool-2-thread-4进来了.跑的任务是 3 执行时间是 2 当前时间是 2022-03-03T21:45:20.336
         pool-2-thread-2进来了.跑的任务是 0 执行时间是 4 当前时间是 2022-03-03T21:45:20.337
         pool-2-thread-1进来了.跑的任务是 4 执行时间是 2 当前时间是 2022-03-03T21:45:21.336
         pool-2-thread-3进来了.跑的任务是 2 执行时间是 5 当前时间是 2022-03-03T21:45:22.337
         pool-2-thread-2进来了.跑的任务是 7 执行时间是 2 当前时间是 2022-03-03T21:45:22.337
         pool-2-thread-4进来了.跑的任务是 6 执行时间是 3 当前时间是 2022-03-03T21:45:23.337
         pool-2-thread-1进来了.跑的任务是 8 执行时间是 2 当前时间是 2022-03-03T21:45:23.337
         pool-2-thread-3进来了.跑的任务是 9 执行时间是 2 当前时间是 2022-03-03T21:45:24.337
         pool-2-thread-5进来了.跑的任务是 5 执行时间是 5 当前时间是 2022-03-03T21:45:24.337
         pool-2-thread-4进来了.跑的任务是 1 执行时间是 3 当前时间是 2022-03-03T21:45:26.337
         pool-2-thread-2进来了.跑的任务是 0 执行时间是 4 当前时间是 2022-03-03T21:45:26.337
         pool-2-thread-5进来了.跑的任务是 4 执行时间是 2 当前时间是 2022-03-03T21:45:26.338
         pool-2-thread-1进来了.跑的任务是 2 执行时间是 4 当前时间是 2022-03-03T21:45:27.337
         */
        for (int i = 0; i < 10; i++ ) {
//            scheduledThreadPoolExecutor.schedule(thread, 5, TimeUnit.SECONDS);

            scheduledThreadPoolExecutor.scheduleAtFixedRate(new MyThread(i), 2, 3, TimeUnit.SECONDS);
        }

        Thread.currentThread().join();

    }

}



```

### Java线程的状态

以下是这些状态的总结，状态的的英文名字对应像jstack或者visualvm中的线程名字

1. 初始(NEW)：新创建了一个线程对象，但还没有调用start()方法。
2. 运行(RUNNABLE)：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”。
线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取CPU的使用权，此时处于就绪状态（ready）。就绪状态的线程在获得CPU时间片后变为运行中状态（running）。
3. 阻塞(BLOCKED)：表示线程阻塞于锁。
4. 等待(WAITING)：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。
5. 超时等待(TIMED_WAITING)：该状态不同于WAITING，它可以在指定的时间后自行返回。
6. 终止(TERMINATED)：表示该线程已经执行完毕。

下面这个转换图很重要

![threadChange]({{ "/assets/img/javathread/threadChange.jpeg" | relative_url}})

Thread类和VisualVM中的对于应关系

| Thread类 | VisualVM |
| :----: | :----:  |
| RUNNABLE | 运行 |
| TIMED_WAITING(sleeping) | 休眠 |
| TIMED_WAITING(on object monitor) WAITING(on object monitor) | 等待 |
| TIMED_WAITING(parking) WAITING(parking) | 驻留 |
| BLOCKED(on object monitor) | 监视 |

1、初始状态
实现Runnable接口和继承Thread可以得到一个线程类，new一个实例出来，线程就进入了初始状态。

2.1、就绪状态
就绪状态只是说你资格运行，调度程序没有挑选到你，你就永远是就绪状态。
调用线程的start()方法，此线程进入就绪状态。
当前线程sleep()方法结束，其他线程join()结束，等待用户输入完毕，某个线程拿到对象锁，这些线程也将进入就绪状态。
当前线程时间片用完了，调用当前线程的yield()方法，当前线程进入就绪状态。
锁池里的线程拿到对象锁后，进入就绪状态。
2.2、运行中状态
线程调度程序从可运行池中选择一个线程作为当前线程时线程所处的状态。这也是线程进入运行状态的唯一一种方式。

3、阻塞状态
阻塞状态是线程阻塞在进入synchronized关键字修饰的方法或代码块(获取锁)时的状态。

4、等待
处于这种状态的线程不会被分配CPU执行时间，它们要等待被显式地唤醒，否则会处于无限期等待的状态。

5、超时等待
处于这种状态的线程不会被分配CPU执行时间，不过无须无限期等待被其他线程显示地唤醒，在达到一定时间后它们会自动唤醒。

6、终止状态
当线程的run()方法完成时，或者主线程的main()方法完成时，我们就认为它终止了。这个线程对象也许是活的，但是，它已经不是一个单独执行的线程。线程一旦终止了，就不能复生。
在一个终止的线程上调用start()方法，会抛出java.lang.IllegalThreadStateException异常。
等待队列

给一个转换的例子

![threadSample]({{ "/assets/img/javathread/threadsample.jpg" | relative_url}})

同步队列状态
当前线程想调用对象A的同步方法时，发现对象A的锁被别的线程占有，此时当前线程进入同步队列。简言之，同步队列里面放的都是想争夺对象锁的线程。
当一个线程1被另外一个线程2唤醒时，1线程进入同步队列，去争夺对象锁。
同步队列是在同步的环境下才有的概念，一个对象对应一个同步队列。
线程等待时间到了或被notify/notifyAll唤醒后，会进入同步队列竞争锁，如果获得锁，进入RUNNABLE状态，否则进入BLOCKED状态等待获取锁。

几个方法的比较
Thread.sleep(long millis)，一定是当前线程调用此方法，当前线程进入TIMED_WAITING状态，但不释放对象锁，millis后线程自动苏醒进入就绪状态。作用：给其它线程执行机会的最佳方式。
Thread.yield()，一定是当前线程调用此方法，当前线程放弃获取的CPU时间片，但不释放锁资源，由运行状态变为就绪状态，让OS再次选择线程。作用：让相同优先级的线程轮流执行，但并不保证一定会轮流执行。实际中无法保证yield()达到让步目的，因为让步的线程还有可能被线程调度程序再次选中。Thread.yield()不会导致阻塞。该方法与sleep()类似，只是不能由用户指定暂停多长时间。
thread.join()/thread.join(long millis)，当前线程里调用其它线程t的join方法，当前线程进入WAITING/TIMED_WAITING状态，当前线程不会释放已经持有的对象锁。线程t执行完毕或者millis时间到，当前线程一般情况下进入RUNNABLE状态，也有可能进入BLOCKED状态（因为join是基于wait实现的）。
obj.wait()，当前线程调用对象的wait()方法，当前线程释放对象锁，进入等待队列。依靠notify()/notifyAll()唤醒或者wait(long timeout) timeout时间到自动唤醒。
obj.notify()唤醒在此对象监视器上等待的单个线程，选择是任意性的。notifyAll()唤醒在此对象监视器上等待的所有线程。
LockSupport.park()/LockSupport.parkNanos(long nanos),LockSupport.parkUntil(long deadlines), 当前线程进入WAITING/TIMED_WAITING状态。对比wait方法,不需要获得锁就可以让线程进入WAITING/TIMED_WAITING状态，需要通过LockSupport.unpark(Thread thread)唤醒。


### ThreadLocal和InheritableThreadLocal

ThreadLocal其实就是每个线程中有一块属于自己的变量。之前会觉得ThreadLocal是不是就是TLAB，但应该不是

看看ThreadLocal的实现细节：首先，我们需要new一个ThreadLocal对象，那么ThreadLocal的构造函数做了什么呢？

```java
/**
     * Creates a thread local variable.
     * @see #withInitial(java.util.function.Supplier)
     */
    public ThreadLocal() {
    }
```
很遗憾它什么都没做，那么初始化的过程势必是在首次set的时候做的，我们来看一下set方法的细节：

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

看起来首先根据当前线程获取到了一个ThreadLocalMap，getMap方法是做了什么？

```java
ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
非常的简洁，是和Thread与生俱来的，我们看一下Thread中的相关定义：

```java
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
获得了线程的ThreadLocalMap之后，如果不为null，说明不是首次set，直接set就可以了，注意key是this，也就是当前的ThreadLocal啊不是Thread。如果为空呢？说明还没有初始化，那么就需要执行createMap这个方法：
```java
void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
没什么特别的，就是初始化线程的threadLocals，然后设定key-value。下面分析一下get的逻辑：

```java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
和set一样，首先根据当前线程获取ThreadLocalMap，然后判断是否为null，如果为null，说明ThreadLocalMap还没有被初始化啊，那么就返回方法setInitialValue的结果，这个方法做了什么？

```java
private T setInitialValue() {
			 T value = initialValue();
			 Thread t = Thread.currentThread();
			 ThreadLocalMap map = getMap(t);
			 if (map != null)
					 map.set(this, value);
			 else
					 createMap(t, value);
			 return value;
	 }

	 protected T initialValue() {
			 return null;
	 }
```
最后会返回null，但是会做一些初始化的工作，和set一样。在get里面，如果返回的ThreadLocalMap不为null，则说明ThreadLocalMap已经被初始化了，那么就可以正常根据ThreadLocal作为key获取了。

当线程退出时，会清理ThreadLocal，可以看下面的代码：

```java
/**
     * This method is called by the system to give a Thread
     * a chance to clean up before it actually exits.
     */
    private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
```

这里做了大量“Help GC”的工作。如果我们想要显示的清理ThreadLocal，可以使用remove方法：

```java
public void remove() {
				ThreadLocalMap m = getMap(Thread.currentThread());
				if (m != null)
						m.remove(this);
		}
```

而InheritableThreadLocal是个啥呢，ThreadLocal固然很好，但是子线程并不能取到父线程的ThreadLocal的变量，比如下面的代码：

```java
private static ThreadLocal<Integer> integerThreadLocal = new ThreadLocal<>();
	 private static InheritableThreadLocal<Integer> inheritableThreadLocal =
					 new InheritableThreadLocal<>();

	 public static void main(String[] args) throws InterruptedException {

			 integerThreadLocal.set(1001); // father
			 inheritableThreadLocal.set(1002); // father

			 new Thread(() -> System.out.println(Thread.currentThread().getName() + ":"
							 + integerThreadLocal.get() + "/"
							 + inheritableThreadLocal.get())).start();

	 }

//output:
Thread-0:null/1002
```
使用ThreadLocal不能继承父线程的ThreadLocal的内容，而使用InheritableThreadLocal时可以做到的，这就可以很好的在父子线程之间传递数据了。

InheritableThreadLocal继承了ThreadLocal，然后重写了上面三个方法，所以除了上面三个方法之外，其他所有对InheritableThreadLocal的调用都是对ThreadLocal的调用，没有什么特别的。我们上文中提到了Thread类，里面有我们本文关心的两个成员，我们来看一下再Thread中做了哪些工作，我们跟踪一下new一个Thread的调用路径：

```java
new Thread()

init(ThreadGroup g, Runnable target, String name, long stackSize)


init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals)

->
           if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);

createInheritedMap(ThreadLocalMap parentMap)


ThreadLocalMap(ThreadLocalMap parentMap)
```
上面列出了最为关键的代码，可以看到，最后会调用ThreadLocal的createInheritedMap方法，而该方法会新建一个ThreadLocalMap，看一下构造函数的内容：

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (int j = 0; j < len; j++) {
                Entry e = parentTable[j];
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
```

parentMap就是父线程的ThreadLocalMap，这个构造函数的意思大概就是将父线程的ThreadLocalMap复制到自己的ThreadLocalMap里面来，这样我们就可以使用InheritableThreadLocal访问到父线程中的变量了。


### 一些个人的理解

同步与异步
同步：在发出一个调用时，在没有得到结果之前，该调用就不返回。一旦返回，必然会得到返回值。
异步：在调用发出之后，这个调用就直接返回。随后，被调用者通过状态、通知来通知调用者，或者通过回调函数来处理调用。
同步与异步关注的是消息通信机制。

阻塞与非阻塞
阻塞：调用结果返回之前，当前线程会被挂起。调用线程只有在得到结果之后才会返回。
非阻塞：调用在不能立刻得到结果之前，该调用不会阻塞当前线程，当前线程仍会处理其他调用。
阻塞与非阻塞关注的是程序在等待调用结果（消息，返回值）时的状态。

并行与并发
并发：在同一个处理器上“同时”处理多个任务。通过cpu调度算法，让用户看上去是同时处理。
并行：在多台机器上同时处理多个任务，真正的同时。

并发编程三个特征
原子性：类似于数据库事务，要么全部执行，要么不执行。
如：num++不具有原子性，先取出num的值，再进行加1。
而a=1,return a则都具有原子性。
可见性：某个线程对共享变量做了修改后，其他线程可以立马感知到该变量的修改。
有序性：若在本线程内观察，所有操作都是有序的；若在一个线程中观察其他线程，所有的操作都是无序的。前半句指“线程内表现为串行语义”，后半句指“指令重排序”现象和“工作内存中主内存同步延迟”现象。

Sychronized与Volatile
Sychronized：可以保证原子性、可见性和有序性。
Sychronized关键字能保证在同一时刻，只有一个线程可以获取锁执行同步代码，执行完之后释放锁之前，会将修改后变量的值刷新的主内存中。
Volatile：可以保证可见性和有序性，不能保证操作的原子性。
被Volatile关键字修饰的变量，在写操作后会加入一条store指令，强行将共享变量最新的值刷新到主内存中。在读操作前，会加入一条load指令，强行从主内存中读取共享变量最新的值。

Java中的锁
Sychronized
ReentrantLock
ReentrantReadWriteLock
这三种锁是怎么实现的？什么是AQS？什么是CAS？CAS的ABA问题怎么解决？集群环境下如何实现同步？来看下

看下这篇文章[写的不错](https://www.cnblogs.com/lifegoeson/p/13683785.html)

CAS在后面JVM的文章中也有所介绍，CAS时Compare And Swap缩写，即比较与交换是用于实现多线程同步的原子指令，它将内存位置的内容与给定值相比较，相同则修改内存位置的值为新值，而整个操作是调用的UnSafe的compareAndSwapObject、compareAndSwapInt或者compareAndSwapLong完成的，而这些方法都是native修饰的本地方法，是一种系统原语系统支持的操作

CAS是一种无锁对象的原子操作，锁分为乐观锁和悲观锁，乐观派抱着几乎不会发生修改同一资源的状态，任意操作同意对象资源，如果遇到修改同一资源的情况，资源不会修改成功，能够保证资源的安全，而悲观派会认为同一资源被错误修改后会造成不可挽回的局面，故自能有一个线程修改资源，这样总会对系统性能产生一定的影响，拖慢自行速度，CAS即无锁执行者，被CAS修饰过的资源可以同时被多个线程修改依然能保证系统安全，无锁不需要等待提高系统性能，jdk提供的CAS原理实现的并发类Automic系列运用及其原理介绍。

首先介绍java的指针操作类UnSafe，Unsafe类是在sun.misc包下，不属于Java标准。但是很多Java的基础类库，包括一些被广泛使用的高性能开发库都是基于Unsafe类开发的，因为UnSafe使Java像C语言一样使其拥有操作内存指针的能力，因为操作内存指针容易出错，故起名UnSafe不安全的类，因此Java官方并不建议使用的，但CAS原理就是UnSafe类中的compareAndSwapObject、compareAndSwapInt和compareAndSwapLong方法实现的，该方法需传入四个参数：第一个参数代表给定的对象，第二个参数代表给定对象再内存中的偏移量，第三个参数标识对象的期望值，第四个参数标识要修改的值，并发包中的Automic系列的原子操作类都是使用UnSafe类实现的。UnSafe的源码如下

```java
/**
* 第一个参数var1代表给定对象，第二个参数var2代表var1对象在内存中的偏移量，第三个参数var3为期望修改* 的对象旧值，第四个参数var4代表要修改的值或着说是修改后的值。
**/
public final native boolean compareAndSwapObject(Object var1, long var2, Object var3, Object var4);

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);

```
举例AtomicInteger源码实现原理，其中的getAndSet
```java
package java.util.concurrent.atomic;
import java.util.function.IntUnaryOperator;
import java.util.function.IntBinaryOperator;
import sun.misc.Unsafe;

public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // 获取UnSafe对象实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    //对象在内存中的偏移量
    private static final long valueOffset;

    //初始化valueOffset
    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    //对象属性值
    private volatile int value;

    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    public AtomicInteger() {
    }

    /**
    * 调用的UnSafe的getAndSetInt方法，给定值和偏移量和修改的值，
    * 获取修改的值var5作为compareAndSwapInt的第三个参数用来和var1比较相同则执行更新操作
    * while循环知道操作成功。
    *public final int getAndSetInt(Object var1, long var2, int var4) {
    *   int var5;
    *   do {
    *       var5 = this.getIntVolatile(var1, var2);
    *   } while(!this.compareAndSwapInt(var1, var2, var5, var4));
    *
    *    return var5;
    *}
    **/
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

    //调用UnSafe的compareAndSwapInt方法保证CAS
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    //调用UnSafe的compareAndSwapInt方法保证CAS
    public final boolean weakCompareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

    //调用UnSafe的getAndAddInt再调用UnSafe的getAndSetInt方法保证CAS
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }


    public final int getAndDecrement() {
        return unsafe.getAndAddInt(this, valueOffset, -1);
    }

    .........
 }
```
主要看下这边

```java
/**
    * 调用的UnSafe的getAndSetInt方法，给定值和偏移量和修改的值，
    * 获取修改的值var5作为compareAndSwapInt的第三个参数用来和var1比较相同则执行更新操作
    * while循环知道操作成功。
    *public final int getAndSetInt(Object var1, long var2, int var4) {
    *   int var5;
    *   do {
    *       var5 = this.getIntVolatile(var1, var2);
    *   } while(!this.compareAndSwapInt(var1, var2, var5, var4));
    *
    *    return var5;
    *}
    **/
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }

```

那`AtomicInteger`在没有锁的情况下是如何做到数据正确性的捏，就是借助了`volatile`原语，保证线程间的数据是可见的（共享的）。

```java
private volatile int value;

public final int get() {
    return value;
}
```

来看下++i是怎么做到的

```java
public final int incrementAndGet() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return next;
    }
}
```
在这里采用了CAS操作，每次从内存中读取数据然后将此数据和+1后的结果进行CAS操作，如果成功就返回结果，否则重试直到成功为止。再看看上面`compareAndSet`的操作，又用到了Unsafe，利用JNI完成CPU指令操作。因此对于synchronized阻塞算法，J.U.C在性能上有了很大的提升


CAS会导致一个ABA的问题，操作对象，获取对象后，执行CAS操作前，被其他线程修改后，且又修改为原来的对象值，导致CAS忽略其他线程的修改，成功执行CAS对象修改，这种情况就叫做ABA问题。举个例子说明

栈中有两个变量 A B，A是栈顶，线程1要将栈顶改成B，所以类似`head.compareAndSet(A, B)`，但在线程1执行这条指令之前，线程2介入，将A，B都出栈，又将D，C，A入栈，此时栈里面的情况是 A C D，而B变成了游离态，此时又轮到线程1去执行CAS操作了，检测发现栈顶还是A，CAS成功，栈顶变成了B，但实际上B.next已经是null了，那么C，D就被平白无故的丢弃了，这就是ABA带来的问题

各种乐观锁的实现中通常都会用版本戳version来对记录或对象标记，避免并发操作带来的问题，在Java中，AtomicStampedReference<E>也实现了这个作用，它通过包装[E,Integer]的元组来对对象标记版本戳stamp，从而避免ABA问题，例如下面的代码分别用AtomicInteger和AtomicStampedReference来对初始值为100的原子整型变量进行更新，AtomicInteger会成功执行CAS操作，而加上版本戳的AtomicStampedReference对于ABA问题会执行CAS失败：

```java
package com.qiming.concurrent.atomic;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.concurrent.atomic.AtomicStampedReference;

/**
 * ABA的一个模拟和解决
 */
public class ABA {

  private static AtomicInteger atomicInt = new AtomicInteger(100);
  //这个可以解决ABA问题
  private static AtomicStampedReference<Integer> atomicStampedRef =
      new AtomicStampedReference<Integer>(100, 0);

  public static void main(String[] args) throws InterruptedException {
    Thread intT1 = new Thread(new Runnable() {
      @Override
      public void run() {
        atomicInt.compareAndSet(100, 101);
        atomicInt.compareAndSet(101, 100);
      }
    });

    Thread intT2 = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        boolean c3 = atomicInt.compareAndSet(100, 101);
        System.out.println(c3);        //true
      }
    });

    intT1.start();
    intT2.start();
    intT1.join();
    intT2.join();

    Thread refT1 = new Thread(new Runnable() {
      @Override
      public void run() {
        try {
          TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        atomicStampedRef.compareAndSet(100, 101,
            atomicStampedRef.getStamp(), atomicStampedRef.getStamp()+1);
        atomicStampedRef.compareAndSet(101, 100,
            atomicStampedRef.getStamp(), atomicStampedRef.getStamp()+1);
      }
    });

    Thread refT2 = new Thread(new Runnable() {
      @Override
      public void run() {
        int stamp = atomicStampedRef.getStamp();
        System.out.println("before sleep : stamp = " + stamp);    // stamp = 0
        try {
          TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        System.out.println("after sleep : stamp = " + atomicStampedRef.getStamp());//stamp = 1
        boolean c3 = atomicStampedRef.compareAndSet(100, 101, stamp, stamp+1);
        System.out.println(c3);        //false
      }
    });

    refT1.start();
    refT2.start();
  }


}
```

ABA的场景

它的执行流程如下：

线程一：取款，获取原值 200 元，与 200 元比对成功，减去 100 元，修改结果为 100 元。

线程二：取款，获取原值 200 元，阻塞等待修改。

线程三：转账，获取原值 100 元，与 100 元比对成功，加上 100 元，修改结果为 200 元。

线程二：取款，恢复执行，原值为 200 元，与 200 元对比成功，减去 100 元，修改结果为 100 元。

最终结果是100元，不符合实际，线程一和线程二相当于同时进行的


再说下AQS

AQS是AbstractQueuedSynchronizer的简称（队列同步器），它是一个Java提供的底层同步工具类，用一个int类型的变量表示同步状态，并提供了一系列的CAS操作来管理这个同步状态。AQS的主要作用是为Java中的并发同步组件提供统一的底层支持，例如ReentrantLock，CountdowLatch就是基于AQS实现的，用法是通过继承AQS实现其模版方法，然后将子类作为同步组件的内部类。

同步队列是AQS很重要的组成部分，它是一个双端队列，遵循FIFO原则，主要作用是用来存放在锁上阻塞的线程，当一个线程尝试获取锁时，如果已经被占用，那么当前线程就会被构造成一个Node节点加入到同步队列的尾部，队列的头节点是成功获取锁的节点，当头节点线程释放锁时，会唤醒后面的节点并释放当前头节点的引用。

想象一个队列

**独占锁的获取和释放流程**
获取

调用入口方法acquire(arg)
调用模版方法tryAcquire(arg)尝试获取锁，若成功则返回，若失败则走下一步
将当前线程构造成一个Node节点，并利用CAS将其加入到同步队列到尾部，然后该节点对应到线程进入自旋状态
自旋时，首先判断其前驱节点释放为头节点&是否成功获取同步状态，两个条件都成立，则将当前线程的节点设置为头节点，如果不是，则利用LockSupport.park(this)将当前线程挂起 ,等待被前驱节点唤醒

释放

调用入口方法release(arg)
调用模版方法tryRelease(arg)释放同步状态
获取当前节点的下一个节点
利用LockSupport.unpark(currentNode.next.thread)唤醒后继节点（接获取的第四步）

**共享锁的获取和释放流程**

获取
调用acquireShared(arg)入口方法
进入tryAcquireShared(arg)模版方法获取同步状态，如果返返回值>=0，则说明同步状态(state)有剩余，获取锁成功直接返回
如果tryAcquireShared(arg)返回值<0，说明获取同步状态失败，向队列尾部添加一个共享类型的Node节点，随即该节点进入自旋状态
自旋时，首先检查前驱节点释放为头节点&tryAcquireShared()是否>=0(即成功获取同步状态)
如果是，则说明当前节点可执行，同时把当前节点设置为头节点，并且唤醒所有后继节点
如果否，则利用LockSupport.unpark(this)挂起当前线程，等待被前驱节点唤醒

释放

调用releaseShared(arg)模版方法释放同步状态
如果释放成，则遍历整个队列，利用LockSupport.unpark(nextNode.thread)唤醒所有后继节点

独占锁和共享锁设计上的区别

独占锁的同步状态值为1，即同一时刻只能有一个线程成功获取同步状态
共享锁的同步状态>1，取值由上层同步组件确定
独占锁队列中头节点运行完成后释放它的直接后继节点
共享锁队列中头节点运行完成后释放它后面的所有节点
共享锁中会出现多个线程（即同步队列中的节点）同时成功获取同步状态的情况

重入锁指的是当前线程成功获取锁后，如果再次访问该临界区，则不会对自己产生互斥行为。Java中对ReentrantLock和synchronized都是可重入锁，synchronized由jvm实现可重入，ReentrantLock都可重入性基于AQS实现。重入锁的基本原理是判断上次获取锁的线程是否为当前线程，如果是则可再次进入临界区，如果不是，则阻塞。所以**重入锁的最主要逻辑就锁判断上次获取锁的线程是否为当前线程**。

Java提供了一个基于AQS到读写锁实现ReentrantReadWriteLock，该读写锁到实现原理是：将同步变量state按照高16位和低16位进行拆分，高16位表示读锁，低16位表示写锁。

小补充下，很多人说的JUC，其实就是java.util.concurrent包，关于java的并发内容的

![juc]({{ "/assets/img/javathread/juc.png" | relative_url}})

### 多线程的案例

1、本地有很多文件，每个文件里面都有若干单词，用多线程技术去完成各种单词数量的总结

这个需求有IO，有Pattern，有多线程，可以作为练习尝试，本地自己做点测试文件

```java
package com.qiming.ms;

import java.io.BufferedReader;
import java.io.FileNotFoundException;
import java.io.FileReader;
import java.util.Map;
import java.util.Set;
import java.util.TreeMap;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.ThreadPoolExecutor.AbortPolicy;
import java.util.concurrent.TimeUnit;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

/**
 * 本地有很多文件，每个文件里面都有若干单词，用多线程技术去完成各种单词数量的总结
 */

public class CountWords implements Callable{

  private int beginTextNum;
  private int endTextNum;

  public CountWords(int beginTextNum, int endTextNum) {
    this.beginTextNum = beginTextNum;
    this.endTextNum = endTextNum;
  }

  public static void main(String[] args) {

//    try {
//      Thread.sleep(15000);
//    } catch (InterruptedException e) {
//      e.printStackTrace();
//    }

    //构建一个线程池
    ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 20, 1000, TimeUnit.MILLISECONDS, new ArrayBlockingQueue<Runnable>(10),
    Executors.defaultThreadFactory(), new AbortPolicy());

    Map<String, Integer> result = new TreeMap<String, Integer>();
    //如果采用下面的方式，get的时候就会阻塞，那么就是串行的起线程了，相当于一个一个做完
//    for (int i = 0; i < 6; i++) {
//      Future<Map<String, Integer>> future = executor.submit(new CountWords((i*5) + 1, (i*5) + 5));
//      try {
//        Map<String, Integer> map = future.get();
//        mergeMap(result, map);
//      } catch (InterruptedException e) {
//        e.printStackTrace();
//      } catch (ExecutionException e) {
//        e.printStackTrace();
//      }
//    }

    //下面这个方式是并行的线程，返回是无序的，说明是多线程在跑
    Future<Map<String, Integer>> future1 = executor.submit(new CountWords(1, 5));
    Future<Map<String, Integer>> future2 = executor.submit(new CountWords(6, 10));
    Future<Map<String, Integer>> future3 = executor.submit(new CountWords(11, 15));
    Future<Map<String, Integer>> future4 = executor.submit(new CountWords(16, 20));
    Future<Map<String, Integer>> future5 = executor.submit(new CountWords(21, 25));
    Future<Map<String, Integer>> future6 = executor.submit(new CountWords(26, 30));
    try {
      Map<String, Integer> map1 = future1.get();
      Map<String, Integer> map2 = future2.get();
      Map<String, Integer> map3 = future3.get();
      Map<String, Integer> map4 = future4.get();
      Map<String, Integer> map5 = future5.get();
      Map<String, Integer> map6 = future6.get();
      mergeMap(result, map1);
      mergeMap(result, map2);
      mergeMap(result, map3);
      mergeMap(result, map4);
      mergeMap(result, map5);
      mergeMap(result, map6);
    } catch (InterruptedException e) {
      e.printStackTrace();
    } catch (ExecutionException e) {
      e.printStackTrace();
    }

    Set<String> keys = result.keySet();
    for (String key : keys) {
      Integer value = result.get(key);
      System.out.println("所有文本中单词 " + key + " 的次数是" + value);
    }

    executor.shutdown();

//    try {
//      Thread.sleep(Integer.MAX_VALUE);
//    } catch (InterruptedException e) {
//      e.printStackTrace();
//    }
  }

  @Override
  public Map<String, Integer> call() throws Exception {

    Pattern pattern = Pattern.compile("[a-zA-Z']+"); //a到z或A到Z
    Map<String, Integer> result = new TreeMap<String, Integer>();
    for (int i = beginTextNum; i <= endTextNum; i++) {
      BufferedReader br = null;
      StringBuilder sb = new StringBuilder();
      String filePath = "E:\\Code\\TestDataSample\\word" + i + ".txt";
      try {
        br = new BufferedReader(new FileReader(filePath));
        String line = null;
        while ((line = br.readLine()) != null) {
          sb = sb.append(line);
        }
      } catch (FileNotFoundException e) {
        e.printStackTrace();
      } finally {
        br.close();
      }

      String sbLowCase = sb.toString().toLowerCase();
      Matcher matcher = pattern.matcher(sbLowCase);

      //单词的总数
      int total = 0;
      Map<String, Integer> map = new TreeMap<String, Integer>();
      while (matcher.find()) {
        String word = matcher.group();
        total++;
        //Integer用null
        Integer num = null;
        if (map.containsKey(word)) {
          num = map.get(word);
          num += 1;
        } else {
          num = 1;
        }
        map.put(word, num);
      }

      Set<String> keys = map.keySet();
      for (String key : keys) {
        Integer value = map.get(key);
        System.out.println("第" + i +"个文本中单词 " + key + " 的次数是" + value);
      }
      System.out.println("第" + i + "个文本中单词总数是 " + total);
      System.out.println("第" + i + "个文本中不重复的单词总数是 " + map.size());

      //合并一下
      mergeMap(result, map);
    }
    return result;
  }

  public static void mergeMap(Map<String, Integer> base, Map<String, Integer> map) {
    Set<String> set = map.keySet();
    for (String key : set) {
      Integer value = null;
      if (base.containsKey(key)) {
        Integer newValue = map.get(key);
        Integer oldValue = base.get(key);
        value = newValue + oldValue;
      } else {
        value = map.get(key);
      }
      base.put(key, value);
    }
  }
}

```
