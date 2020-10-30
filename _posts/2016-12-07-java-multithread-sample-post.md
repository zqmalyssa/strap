---
layout: post
title: 多线程案例
tags: [code, java]
author-id: zqmalyssa
---

专门总结一些多线程的案例吧

#### 自己实现一个读写锁

具体如下，分成锁的设计，读线程，写线程

```java
package com.qiming.algorithm.multithread;


import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.PriorityBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * 使用线程池设计读写锁
 *
 * 与传统锁不同的是读写锁的规则是可以共享读，但只能一个写，总结起来为：读读不互斥，读写互斥，写写互斥，而一般的独占锁是：
 * 读读互斥，读写互斥，写写互斥，而场景中往往读远远大于写，读写锁就是为了这种优化而创建出来的一种机制。注意是读远远大于写
 *
 *
 *
 */
public class UseExecutor {

  private static ExecutorService pool;
  private static int number = 0;

  private final MyReadWriterLock lock = new MyReadWriterLock();

  //创建一个共享数据对象，用于读写操作
  private final char[] buffer;

  public UseExecutor(int size) {
    this.buffer = new char[size];
    for (int i = 0; i < buffer.length; i++) {
      buffer[i] = '*';
    }
  }

  public char[] read() {
    try {
      lock.lockRead();
      return this.doRead();
    } catch (InterruptedException e) {
      e.printStackTrace();
    } finally {
      lock.unlockRead();
    }
    return new char[0];
  }

  public void write(char c) {
    try {
      lock.lockWriter();
      this.doWrite(c);
    } catch (InterruptedException e) {
      e.printStackTrace();
    } finally {
      lock.unlockWriter();
    }
  }

  private void doWrite(char c) {
    for (int i = 0; i < buffer.length; i++) {
      buffer[i] = c;
      slowly(10);
    }
  }

  private char[] doRead() {
    char[] newBuf = new char[buffer.length];
    for (int i = 0; i < buffer.length; i++) {
      newBuf[i] = buffer[i];
    }
    slowly(500);
    return newBuf;
  }

  private void slowly(int ms) {
    try {
      Thread.sleep(ms);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

//  public static void main(String[] args) {
//
//    pool = new ThreadPoolExecutor(10, 20, 1000, TimeUnit.MILLISECONDS, new PriorityBlockingQueue<Runnable>(),
//        Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
//
//    //三个读线程
//    for (int i = 0; i < 3; i++) {
//      pool.execute(new Runnable() {
//        @Override
//        public void run() {
//          try {
//            lock.lockRead();
//            System.out.println("我是读线程" + Thread.currentThread().getName() + "目前的number值是" +  + number);
//          } catch (Exception e){
//            e.printStackTrace();
//          } finally {
//            lock.unlockRead();
//          }
//        }
//      });
//    }
//
//
//    //一个写线程
//    for (int i = 0; i < 3; i++) {
//      pool.execute(new Runnable() {
//        @Override
//        public void run() {
//          try {
//            lock.lockWriter();
//            number--;
//            System.out.println("我是写线程" + Thread.currentThread().getName() + "目前的number值是" +  + number);
//          } catch (Exception e){
//            e.printStackTrace();
//          } finally {
//            lock.unlockWriter();
//          }
//        }
//      });
//    }
//
//
//    pool.shutdown();
//  }

}

/**
 * 模拟读写锁
 */
class MyReadWriterLock {

  /**
   * 读锁持有个数
   */
  private volatile int read;
  /**
   * 写锁持有个数
   */
  private volatile int write;

  public MyReadWriterLock() {
    this.read = 0;
    this.write = 0;
  }

  /**
   * 获取读锁，读锁在写锁不存在的时候才能获取
   * @throws InterruptedException
   */
  public synchronized void lockRead() throws InterruptedException {
    while (write > 0) {
      this.wait();
    }
    this.read++;
  }

  /**
   * 释放读锁
   */
  public synchronized void unlockRead() {
    read--;
    this.notifyAll();
  }

  /**
   * 获取写锁，当读锁，写锁都有时，wait()
   * @throws InterruptedException
   */
  public synchronized void lockWriter() throws InterruptedException{
    while (read > 0 || write > 0) {
      this.wait();
    }
    this.write++;
  }

  /**
   * 释放写锁
   */
  public synchronized void unlockWriter() {
    this.write--;
    this.notifyAll();
  }


}

```

读线程

```java
package com.qiming.algorithm.multithread;

public class ReadWorker extends Thread{

  private final UseExecutor useExecutor;

  public ReadWorker(UseExecutor useExecutor) {
    this.useExecutor = useExecutor;
  }

  @Override
  public void run() {
    while (true) {
      char[] readBuffer = useExecutor.read();
      System.out.println(Thread.currentThread().getName() + " reads " + String.valueOf(readBuffer));
    }
  }
}

```

写线程

```java
package com.qiming.algorithm.multithread;

import java.util.Random;

public class WriterWork extends Thread {

  private static final Random random = new Random(System.currentTimeMillis());

  private final UseExecutor useExecutor;
  private final String filler;
  private int index;

  public WriterWork(UseExecutor useExecutor, String filler) {
    this.useExecutor = useExecutor;
    this.filler = filler;
  }


  @Override
  public void run() {
    try {
      while (true) {
        char c = nextChar();
        useExecutor.write(c);
        Thread.sleep(random.nextInt(1000));
      }
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
  }

  public char nextChar() {
    char c = filler.charAt(index);
    index++;
    if (index >= filler.length()) {
      index = 0;
    }
    return c;
  }
}

```
测试一下，最好用线程池启动，记得关闭

```java
package com.qiming.algorithm.multithread;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.PriorityBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ReadWriteLockClient {

  /**
   * 使用线程池
   */
  private static ExecutorService pool;

  public static void main(String[] args) {

    pool = new ThreadPoolExecutor(10, 20, 1000, TimeUnit.MILLISECONDS, new PriorityBlockingQueue<Runnable>(),
        Executors.defaultThreadFactory(), new ThreadPoolExecutor.AbortPolicy());
    final UseExecutor useExecutor= new UseExecutor(10);
//    new ReadWorker(useExecutor).start();
//    new ReadWorker(useExecutor).start();
//    new ReadWorker(useExecutor).start();
//    new ReadWorker(useExecutor).start();
//    new ReadWorker(useExecutor).start();
//
//    new WriterWork(useExecutor, "ddjifjsidjfisd").start();
//    new WriterWork(useExecutor, "DDJIFJSIDJFISD").start();

    //启动多个读线程
    for (int i = 0; i < 5; i++) {
      pool.execute(new ReadWorker(useExecutor));
    }

    //启动较少的写线程
    pool.execute(new WriterWork(useExecutor, "ddjifjsidjfisd"));
    pool.execute(new WriterWork(useExecutor, "DDJIFJSIDJFISD"));

    pool.shutdown();
  }

}

```

#### 加锁和不加锁的写法


lock的condition是加锁的线程同步方法，但实际应用中会有性能问题，可以不加锁去写，有volatile写法和Semaphore(模拟锁)写法。要知道线程的东西加锁的是可以转化的

凡是可以用信号量解决的问题，都可以用管程模型（Lock）来解决，但凡用了锁的，都来试试可否变成无锁的

```java
package com.qiming.algorithm.leetcode;

import java.util.concurrent.Semaphore;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 打印零与奇偶数
 *
 * 相同的一个 ZeroEvenOdd 类实例将会传递给三个不同的线程：
 *
 * 线程 A 将调用 zero()，它只输出 0 。
 * 线程 B 将调用 even()，它只输出偶数。
 * 线程 C 将调用 odd()，它只输出奇数。
 *
 * 每个线程都有一个 printNumber 方法来输出一个整数。请修改给出的代码以输出整数序列 010203040506... ，其中序列的长度必须为 2n。
 *
 * 思路，lock的condition正常写，但是会有超时，不加锁去写，有volatile写法和Semaphore(模拟锁)写法
 *
 */
public class PrintZeroEvenOdd {

  private static int n = 5;
  private static int index;

  //0代表zero，1代表odd，2代表偶数
  private static volatile int flag = 0;

  public PrintZeroEvenOdd(int n) {
    this.n = n;
  }


  public static void main(String[] args) {

    Semaphore zero = new Semaphore(1);
    Semaphore odd = new Semaphore(0);
    Semaphore even = new Semaphore(0);

    /**
     * 以下用锁方法会超时
     */
//    final Lock lock = new ReentrantLock();
//
//    final Condition conditionZero = lock.newCondition();
//    final Condition conditionNum = lock.newCondition();
//
//    //打印0的线程
//    Thread threadA = new Thread(new Runnable() {
//      @Override
//      public void run() {
//        try {
//          lock.lock();
//          for (int i = 0; i < n; i++) {
//            while (index % 2 != 0) {
//              conditionZero.await();
//            }
//            System.out.print(0);
//            index++;
//            conditionNum.signalAll();
//          }
//        } catch (Exception e) {
//          e.printStackTrace();
//        } finally {
//          lock.unlock();
//        }
//
//      }
//    }
//    );
//
//    //打印奇数的线程
//    Thread threadB = new Thread(new Runnable() {
//      @Override
//      public void run() {
//        try {
//          lock.lock();
//          for (int i = 1; i <= n; i = i + 1) {
//            while (index % 2 != 1 || (index + 1) % 2 != 0) {
//              conditionNum.await();
//            }
//            System.out.print(i);
//            index++;
//            conditionZero.signal();
//          }
//        } catch (Exception e) {
//          e.printStackTrace();
//        } finally {
//          lock.unlock();
//        }
//      }
//    });
//
//    //打印偶数的线程
//    Thread threadC = new Thread(new Runnable() {
//      @Override
//      public void run() {
//        try {
//          lock.lock();
//          for (int i = 2; i <= n; i = i + 2) {
//            while (index % 2 != 1 || (index + 1) % 2 != 1) {
//              conditionNum.await();
//            }
//            System.out.print(i);
//            index++;
//            conditionZero.signal();
//          }
//        } catch (Exception e) {
//          e.printStackTrace();
//        } finally {
//          lock.unlock();
//        }
//      }
//    });

    /**
     * 没有lock的写法，用到volatile
     */
    //打印0的线程
//    Thread threadA = new Thread(new Runnable() {
//      @Override
//      public void run() {
//        for (int i = 0; i < n; i++) {
//          while (flag != 0) {
//            Thread.yield();
//          }
//          System.out.print(0);
//          //这样想，i只控制0输出后是奇数还是偶数就行了
//          if (i % 2 != 0) {
//            flag = 2;
//          } else {
//            flag = 1;
//          }
//        }
//
//      }
//    }
//    );
//
//    Thread threadB = new Thread(new Runnable() {
//      @Override
//      public void run() {
//        for (int i = 1; i <= n; i= i+2) {
//          while (flag != 1) {
//            Thread.yield();
//          }
//          System.out.print(i);
//          flag = 0;
//        }
//
//      }
//    }
//    );
//
//    Thread threadC = new Thread(new Runnable() {
//      @Override
//      public void run() {
//        for (int i = 2; i <= n; i= i+2) {
//          while (flag != 2) {
//            Thread.yield();
//          }
//          System.out.print(i);
//          flag = 0;
//        }
//
//      }
//    }
//    );

    /**
     * 使用信号量的方法
     */
    //打印0的线程
    Thread threadA = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 0; i < n; i++) {
          try {
            zero.acquire();
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
          System.out.print(0);
          //这样想，i只控制0输出后是奇数还是偶数就行了
          if (i % 2 != 0) {
            even.release();
          } else {
            odd.release();
          }
        }

      }
    }
    );

    //打印奇数的线程
    Thread threadB = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 1; i <= n; i= i+2) {
          try {
            odd.acquire();
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
          System.out.print(i);
          zero.release();
        }

      }
    }
    );

    //打印偶数的线程
    Thread threadC = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 2; i <= n; i=i+2) {
          try {
            even.acquire();
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
          System.out.print(i);
          zero.release();
        }

      }
    }
    );

    threadA.setName("打印zero线程");
    threadB.setName("打印奇数线程");
    threadC.setName("打印偶数线程");

    threadA.start();
    threadB.start();
    threadC.start();

  }
}

```

#### 交替打印字符串汇总

编写一个可以从 1 到 n 输出代表这个数字的字符串的程序，但是：如果这个数字可以被 3 整除，输出 "fizz"。如果这个数字可以被 5 整除，输出 "buzz"。如果这个数字可以同时被 3 和 5 整除，输出 "fizzbuzz"。

```java
package com.qiming.algorithm.multithread;

import java.util.concurrent.Semaphore;
import java.util.function.IntConsumer;

/**
 * 交替打印字符串
 *
 * 编写一个可以从 1 到 n 输出代表这个数字的字符串的程序，但是：
 * 如果这个数字可以被 3 整除，输出 "fizz"。
 * 如果这个数字可以被 5 整除，输出 "buzz"。
 * 如果这个数字可以同时被 3 和 5 整除，输出 "fizzbuzz"。
 * 例如，当 n = 15，输出： 1, 2, fizz, 4, buzz, fizz, 7, 8, fizz, buzz, 11, fizz, 13, 14, fizzbuzz。
 *
 * 思路：用信号量去做，注意number这个很关键，自我释放锁
 *
 */
public class FizzBuzzMultithreaded {

  private int n;

  private volatile static int flag = 1;

  private static Semaphore fizzSem = new Semaphore(0);
  private static Semaphore buzzSem = new Semaphore(0);
  private static Semaphore fizzBuzzSem = new Semaphore(0);
  private static Semaphore numSem = new Semaphore(1);

  public FizzBuzzMultithreaded(int n) {
    this.n = n;
  }

  // printFizz.run() outputs "fizz".
  public void fizz(Runnable printFizz) throws InterruptedException {

  }

  // printBuzz.run() outputs "buzz".
  public void buzz(Runnable printBuzz) throws InterruptedException {

  }

  // printFizzBuzz.run() outputs "fizzbuzz".
  public void fizzbuzz(Runnable printFizzBuzz) throws InterruptedException {

  }

  // printNumber.accept(x) outputs "x", where x is an integer.
  public void number(IntConsumer printNumber) throws InterruptedException {

  }

  public static void main(String[] args) {

    int m = 15;

    Thread threadFizz = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 3; i <= m; i = i + 3) {
          if (i % 5 != 0) {
            try {
              fizzSem.acquire();
              System.out.println("fizz");
              numSem.release();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
        }
      }
    });

    Thread threadBuzz = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 5; i <= m; i = i + 5) {
          if (i % 3 != 0) {
            try {
              buzzSem.acquire();
              System.out.println("buzz");
              numSem.release();
            } catch (InterruptedException e) {
              e.printStackTrace();
            }
          }
        }
      }
    });

    Thread threadFizzBuzz = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 15; i <= m; i = i + 15) {
          try {
            fizzBuzzSem.acquire();
            System.out.println("fizzbuzz");
            numSem.release();
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        }
      }
    });

    /**
     * 重点是number这个，有个自我解锁
     */
    Thread threadNumber = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 1; i <= m; i++) {
          try {
            numSem.acquire();
            if (i % 3 == 0 && i % 5 == 0) {
              fizzBuzzSem.release();
            }else if (i % 3 == 0) {
              fizzSem.release();
            }else if (i % 5 == 0) {
              buzzSem.release();
            }else {
              System.out.println(i);
              numSem.release();
            }
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        }
      }
    });

    threadFizz.start();
    threadBuzz.start();
    threadFizzBuzz.start();
    threadNumber.start();
  }

}

```

指定次数的交替打印FooBar，可以用信号量做一波，为啥都是信号量了

```java
package com.qiming.algorithm.multithread;

import java.util.concurrent.Semaphore;

/**
 * 交替打印FooBar
 *
 * 我们提供一个类：
 * class FooBar {
 *   public void foo() {
 *     for (int i = 0; i < n; i++) {
 *       print("foo");
 *     }
 *   }
 *
 *   public void bar() {
 *     for (int i = 0; i < n; i++) {
 *       print("bar");
 *     }
 *   }
 * }
 *
 * 两个不同的线程将会共用一个 FooBar 实例。其中一个线程将会调用 foo() 方法，另一个线程将会调用 bar() 方法。请设计修改程序，以确保 "foobar" 被输出 n 次。
 *
 * 输入: n = 2  输出: "foobarfoobar"  解释: "foobar" 将被输出两次。
 *
 * 思路：信号量做一波
 *
 */
public class PrintFooBarAlternately {

  private int n;

  private Semaphore fooSema = new Semaphore(1);
  private Semaphore barSema = new Semaphore(0);

  public PrintFooBarAlternately(int n) {
    this.n = n;
  }

  public void foo(Runnable printFoo) throws InterruptedException {

    for (int i = 0; i < n; i++) {
      fooSema.acquire();
      // printFoo.run() outputs "foo". Do not change or remove this line.
      printFoo.run();
      barSema.release();
    }
  }

  public void bar(Runnable printBar) throws InterruptedException {

    for (int i = 0; i < n; i++) {
      barSema.acquire();
      // printBar.run() outputs "bar". Do not change or remove this line.
      printBar.run();
      fooSema.release();
    }
  }

}

```
现在有两种线程，氢 oxygen 和氧 hydrogen，你的目标是组织这两种线程来产生水分子。存在一个屏障（barrier）使得每个线程必须等候直到一个完整水分子能够被产生出来。氢和氧线程会被分别给予 releaseHydrogen 和 releaseOxygen 方法来允许它们突破屏障。这些线程应该三三成组突破屏障并能立即组合产生一个水分子。你必须保证产生一个水分子所需线程的结合必须发生在下一个水分子产生之前。换句话说:

如果一个氧线程到达屏障时没有氢线程到达，它必须等候直到两个氢线程到达。
如果一个氢线程到达屏障时没有其它线程到达，它必须等候直到一个氧线程和另一个氢线程到达。

信号量可以多值释放

```java
package com.qiming.algorithm.multithread;

import java.util.concurrent.Semaphore;

/**
 * H2O 生成
 *
 * 现在有两种线程，氢 oxygen 和氧 hydrogen，你的目标是组织这两种线程来产生水分子。存在一个屏障（barrier）使得每个线程必须等候直到一个完整水分子能够被产生出来。
 * 氢和氧线程会被分别给予 releaseHydrogen 和 releaseOxygen 方法来允许它们突破屏障。这些线程应该三三成组突破屏障并能立即组合产生一个水分子。
 * 你必须保证产生一个水分子所需线程的结合必须发生在下一个水分子产生之前。换句话说:
 *
 * 如果一个氧线程到达屏障时没有氢线程到达，它必须等候直到两个氢线程到达。
 * 如果一个氢线程到达屏障时没有其它线程到达，它必须等候直到一个氧线程和另一个氢线程到达。
 * 书写满足这些限制条件的氢、氧线程同步代码。
 *
 * 输入: "HOH"  输出: "HHO"  解释: "HOH" 和 "OHH" 依然都是有效解。
 * 输入: "OOHHHH"  输出: "HHOHHO" 解释: "HOHHHO", "OHHHHO", "HHOHOH", "HOHHOH", "OHHHOH", "HHOOHH", "HOHOHH" 和 "OHHOHH" 依然都是有效解。
 *
 * 思路：信号量，一次性释放多个的操作
 *
 */
public class BuildingH2O {

  private Semaphore h = new Semaphore(2);
  private Semaphore o = new Semaphore(0);

  public BuildingH2O() {

  }

  public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
    h.acquire();
    // releaseHydrogen.run() outputs "H". Do not change or remove this line.
    releaseHydrogen.run();
    o.release();
  }

  public void oxygen(Runnable releaseOxygen) throws InterruptedException {
    o.acquire(2);
    // releaseOxygen.run() outputs "O". Do not change or remove this line.
    releaseOxygen.run();
    h.release(2);
  }

}

```


#### 哲学家进餐问题汇总

设计一个进餐规则（并行算法）使得每个哲学家都不会挨饿；也就是说，在没有人知道别人什么时候想吃东西或思考的情况下，每个哲学家都可以在吃饭和思考之间一直交替下去。5个哲学家5个叉子

一个Semaphore和一个ReentrantLock数组

```java
package com.qiming.algorithm.multithread;

import java.util.concurrent.Semaphore;
import java.util.concurrent.locks.ReentrantLock;

/**
 * 哲学家进餐
 *
 * 5 个沉默寡言的哲学家围坐在圆桌前，每人面前一盘意面。叉子放在哲学家之间的桌面上。（5 个哲学家，5 根叉子），所有的哲学家都只会在思考和进餐两种行为间交替。
 * 哲学家只有同时拿到左边和右边的叉子才能吃到面，而同一根叉子在同一时间只能被一个哲学家使用。每个哲学家吃完面后都需要把叉子放回桌面以供其他哲学家吃面。
 * 只要条件允许，哲学家可以拿起左边或者右边的叉子，但在没有同时拿到左右叉子时不能进食。设计一个进餐规则（并行算法）使得每个哲学家都不会挨饿；也就是说，
 * 在没有人知道别人什么时候想吃东西或思考的情况下，每个哲学家都可以在吃饭和思考之间一直交替下去。
 *
 * 思路，一个Semaphore和一个ReentrantLock数组
 *
 *
 */
public class TheDiningPhilosophers {

  private ReentrantLock[] lockList = {new ReentrantLock(),
      new ReentrantLock(),
      new ReentrantLock(),
      new ReentrantLock(),
      new ReentrantLock()};

  private Semaphore eatLimit = new Semaphore(4);

  public TheDiningPhilosophers() {

  }

  // call the run() method of any runnable to execute its code
  public void wantsToEat(int philosopher,
      Runnable pickLeftFork,
      Runnable pickRightFork,
      Runnable eat,
      Runnable putLeftFork,
      Runnable putRightFork) throws InterruptedException {

    int leftFork = (philosopher + 1) % 5;	//左边的叉子 的编号
    int rightFork = philosopher;	//右边的叉子 的编号

    eatLimit.acquire();	//限制的人数 -1

    lockList[leftFork].lock();	//拿起左边的叉子
    lockList[rightFork].lock();	//拿起右边的叉子

    pickLeftFork.run();	//拿起左边的叉子 的具体执行
    pickRightFork.run();	//拿起右边的叉子 的具体执行

    eat.run();	//吃意大利面 的具体执行

    putLeftFork.run();	//放下左边的叉子 的具体执行
    putRightFork.run();	//放下右边的叉子 的具体执行

    lockList[leftFork].unlock();	//放下左边的叉子
    lockList[rightFork].unlock();	//放下右边的叉子

    eatLimit.release();//限制的人数 +1

  }

}

```

#### 自己实现一个阻塞队列

使用双端列表和线程池自己实现的一个阻塞队列

```java
package com.qiming.algorithm.ms;

import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.SynchronousQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.ThreadPoolExecutor.AbortPolicy;
import java.util.concurrent.TimeUnit;

/**
 * 模拟阻塞队列
 *
 * 生产者和消费者，不要用那种简便的方法做，结合线程池，再结合双端队列
 *
 *
 */
public class TestMain3 {


  private NodeTestMain3 head;
  private NodeTestMain3 tail;

  private Semaphore full;
  private Semaphore empty;
  private Semaphore mutex;

  private static volatile int size;

  public TestMain3(int capacity) {
    this.full = new Semaphore(capacity);
    this.empty = new Semaphore(0);
    this.mutex = new Semaphore(1);

    this.head = new NodeTestMain3(0);
    this.tail = new NodeTestMain3(0);

    this.head.next = tail;
    this.tail.pre = head;
    this.head.pre = null;
    this.tail.next = null;
  }


  public static void main(String[] args) {

    TestMain3 test = new TestMain3(10);

    Producer producer1 = new Producer(test);
    Producer producer2 = new Producer(test);
    Producer producer3 = new Producer(test);

    Consumer consumer1 = new Consumer(test);
    Consumer consumer2 = new Consumer(test);
    Consumer consumer3 = new Consumer(test);

    Thread thread1 = new Thread(new Producer(test), "生产者1");
    Thread thread2 = new Thread(new Producer(test), "生产者2");
    Thread thread3 = new Thread(new Producer(test), "生产者3");

    Thread thread4 = new Thread(new Consumer(test), "消费者1");
    Thread thread5 = new Thread(new Consumer(test), "消费者2");
    Thread thread6 = new Thread(new Consumer(test), "消费者3");

//    new Thread(producer1, "生产者1").start();
//    new Thread(producer2, "生产者2").start();
//    new Thread(producer3, "生产者3").start();
//
//    new Thread(consumer1, "消费者1").start();
//    new Thread(consumer2, "消费者2").start();
//    new Thread(consumer3, "消费者3").start();

    /**
     * 用好线程池
     */
    ThreadPoolExecutor pool = new ThreadPoolExecutor(10, 20, 1000, TimeUnit.MILLISECONDS, new SynchronousQueue<>(),
        Executors.defaultThreadFactory(), new AbortPolicy());

//    pool.execute(producer1);
//    pool.execute(producer2);
//    pool.execute(producer3);
//
//    pool.execute(consumer1);
//    pool.execute(consumer2);
//    pool.execute(consumer3);

    pool.execute(thread1);
    pool.execute(thread2);
    pool.execute(thread3);

    pool.execute(thread4);
    pool.execute(thread5);
    pool.execute(thread6);


    pool.shutdown();

    /**
     * 下面这种最好不要这样写
     */

//    for (int i = 0; i < 12; i++) {
//
//      new Thread(new Runnable() {
//        @Override
//        public void run() {
//          try {
//            test.offer(new NodeTestMain3(0));
//          } catch (InterruptedException e) {
//            System.out.println("不能再放元素了");
//            e.printStackTrace();
//          }
//        }
//      }).start();
//
//    }

//    for (int i = 0; i < 2; i++) {
//
//      new Thread(new Runnable() {
//        @Override
//        public void run() {
//          try {
//            test.poll();
//            System.out.println(Thread.currentThread().getName() + "是poll线程，现在的size是 " + size);
//          } catch (InterruptedException e) {
//            System.out.println("没有新的元素了");
//            e.printStackTrace();
//          }
//        }
//      }).start();
//
//    }

  }

  public void offer(NodeTestMain3 node) throws InterruptedException {

    full.acquire();
    mutex.acquire();

    //插入到链表尾部，先指好该结点的前后，再更新左右两边
    node.pre = tail.pre;
    node.next = tail;

    tail.pre.next = node;
    tail.pre = node;

    size++;
    System.out.println(Thread.currentThread().getName() + " 号调用offer方法，现在的size是 " + size);
    mutex.release();
    empty.release();

  }

  public NodeTestMain3 poll() throws InterruptedException {

    empty.acquire();
    mutex.acquire();

    //删除链表的第一个元素，先要判断，删简单，把两头做绑定
    NodeTestMain3 node = checkPosition(head.next);
    node.pre.next = node.next;
    node.next.pre = node.pre;

    size--;
    System.out.println(Thread.currentThread().getName() + " 号调用poll方法，现在的size是 " + size);
    mutex.release();
    full.release();

    return node;
  }

  private NodeTestMain3 checkPosition(NodeTestMain3 node) {
    if (node == null) {
      throw new RuntimeException("删除的元素是null");
    }
    if (node == head) {
      throw new RuntimeException("删除的元素是头结点，非法");
    }
    if (node == tail) {
      throw new RuntimeException("删除的元素是尾结点，非法");
    }
    return node;
  }


}


class Producer implements Runnable {

  TestMain3 queue;

  public Producer(TestMain3 queue) {
    this.queue = queue;
  }

  @Override
  public void run() {
    int i = 0;
    while (i < 15) {
      NodeTestMain3 node = new NodeTestMain3(0);
      try {
        queue.offer(node);
        i++;
        System.out.println("我是生产者 " + Thread.currentThread().getName() + " 号，这是我生产的第" + i + "个元素");
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }
}


class Consumer implements Runnable {

  TestMain3 queue;

  public Consumer(TestMain3 queue) {
    this.queue = queue;
  }

  @Override
  public void run() {
    int i = 0;
    while (i < 15) {
      try {
        queue.poll();
        i++;
        System.out.println("我是消费者 " + Thread.currentThread().getName() + " 号，这是我消费的第" + i + "个元素");
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
    }
  }
}

class NodeTestMain3 {

  int val;
  NodeTestMain3 pre;
  NodeTestMain3 next;

  public NodeTestMain3(int val) {
    this.val = val;
  }
}

```
