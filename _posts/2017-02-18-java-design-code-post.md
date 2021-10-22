---
layout: post
title: Java Code总结整理
tags: [code, java, design]
author-id: zqmalyssa
---

Java设计模式看完后，再来看看Java Code中一些值得思考和注意的地方

参考《Effective Java》这本书

#### 创建和销毁对象

1. **重叠构造器**当参数有许多时，需要一层一层的写构造器，后面会很难维护，万一颠倒了两个参数，编译器又发现不了，可以使用JavaBeans模式，就是简单的new一个对象，然后使用set方法进行想要值得赋值，但是JavaBean也可能引发"可变"，程序员要付出额外努力确保它线程安全，这边有个模式，`builder模式`，不直接生成想要的方法，用类似set的方法设置参数，最后调用无参的build方法来生成不可变对象(想想openstack4j吧)

```java
package com.qiming.test;

public class BuilderTest {

  private final int size;

  private final int fat;

  private final int sodium;

  private final int service;

  public static class Builder {
    private final int size;
    private final int fat;

    private int sodium = 0;
    private int service = 0;

    public Builder(int size, int fat) {
      this.size = size;
      this.fat = fat;
    }

    public Builder sodium(int sodium) {
      sodium = sodium;
      return this;
    }

    public Builder service(int service){
      service = service;
      return this;
    }

    public BuilderTest build(){
      return new BuilderTest(this);
    }
  }

  private BuilderTest(Builder builder){
    size = builder.size;
    fat = builder.fat;
    sodium = builder.sodium;
    service = builder.service;
  }

  public static void main(String args[]) {
    BuilderTest test = new BuilderTest.Builder(5,10).service(11).sodium(12).build();
  }

}

```
当要再添加参数的时候，就能方便控制了，设置了参数的builder生成了一个很好的抽象工厂，客户可以将这样一个builder传给方法，使该方法能够为客户端创建一个或者多个对象，要用到泛型

```java

public interface Builder<T> {
	public T build();
}

```

2. 单例模式中，先看如下三种方式

```java
package com.qiming.test;

//方式1
public class Elvis {

  public static final Elvis INSTANCE = new Elvis();
  private Elvis() {};

}
```

```java
package com.qiming.test;

//方式2
public class Elvis2 {

  private static final Elvis2 INSTANCE = new Elvis2();
  private Elvis2() {};
  public static Elvis2 getInstance(){return INSTANCE;}

}
```

```java
package com.qiming.test;

//方式3，双重检查枷锁，单例防止线程异常

public class Elvis3 {

  private volatile static Elvis3 unique;

  private Elvis3() {} //注意private的写法

  public static Elvis3 getInstance() { //1
    if (unique == null) { //2
      synchronized (Elvis3.class) { //3
        if (unique == null) { //4
          unique = new Elvis3(); //5
        }
      }
    }
    return unique;
  }

}
```

其中需要volatile关键字的原因是，在并发情况下，如果没有volatile关键字，在第5行会出现问题。unique = new Elvis3();可以分解为3行伪代码
a.memory = allocate() //分配内存
b.ctorInstanc(memory) //初始化对象
c.instance = memory //设置instance指向刚分配的地址

上面的代码在编译运行时，可能会出现重排序从a-b-c排序为a-c-b。在多线程的情况下会出现以下问题。当线程A在执行第5行代码时，B线程进来执行到第2行代码。假设此时A执行的过程中发生了指令重排序，即先执行了a和c，没有执行b。那么由于A线程执行了c导致instance指向了一段地址，所以B线程判断instance不为null，会直接跳到第6行并返回一个未初始化的对象。说的像是有一点道理？？

```java
package com.qiming.test;

//方法4
public enum Elvis4 {

  INSTANCE;

}
```

用枚举，单个元素的枚举类型，很简洁，且面对复杂的序列化和反射攻击时有用，据说是最简洁的单例模式


3. 强化不可实例化


```java
package com.qiming.test;

//想这个类不被实例化，显示写个构造器
public class UtilityClass {

  private UtilityClass() {
    throw new AssertionError();
  }

}
}
```

如果什么不写，会有个默认，公有的构造器，强制写一个，让其不能被实例化，还抛错，也就是内部也不能调用，但是这样做，这个类也不能被子类化了，就当时工具类吧

4. Long和long，虽然可以自动装箱，但是用装箱类型是有代价的，性能会变慢
5. Object中的非final方法(equals, hashcode, toString, clone, finalize)其实都有明确的通用约定，用来被设计成是要覆盖的，覆盖equals方法不要太花哨，就做简单的值比较，否则会导致自反性，传递性等特性的缺失，equals(MyClass o)方法其实是重载了equals(Object o)方法，而不是覆盖！！覆盖equals方法总要覆盖hashcode，相等的对象必须具有相等的散列码！！始终要覆盖toString方法，谨慎的覆盖clone。通用约定如下：

1）在java应用执行期间，只要对象的equals方法的比较操作所用到的信息没有被修改，那么对这同一对象调用多次hashCode方法都必须始终如一地同一个整数。在同一个应用程序的多次执行过程中，每次执行该方法返回的整数可以不一致。

2）如果两个对象根据equals(Object)方法比较是相等的，那么调用这两个对象中任意一个对象的hashCode方法都必须产生同样的整数结果。

3）如果两个对象根据equals(Object)方法比较是不相等的，那么调用这两个对象中任意一个对象的hashCode方法没必要产生不同的整数结果。但是程序猿应该知道，给不同的对象产生截然不同的整数结果，有可能提高散列表（hash table）的性能

**因此，覆盖equals时总是要覆盖hashCode是一种通用的约定，而不是必须的，如果和基于散列的集合（HashMap、HashSet、HashTable）一起工作时，特别是将该对象作为key值的时候，一定要覆盖hashCode，否则会出现错误。那么既然是一种规范，那么作为程序猿的我们就有必要必须执行，以免出现问题。**

为什么覆盖equals的时候要覆盖hashCode？通过HashMap的实现原理，可以看出**当自定义类作为key值存在的时候**一定要这样做，但不作为key值可以选择不这样做（但为了规范起见，还是要覆盖，因此就变成了必须的了）。如果将测试代码中的equals或hashCode注释掉都不能得到正确的结果

这边其实可以根据JDK1.8的hashmap的源代码来分析下原因

总结：put存储过程：将K/V传给put方法时，它调用hashCode计算hash从而得到Node位置，进一步存储，HashMap会根据当前Node的占用情况自动调整容量(超过Load Facotr则resize为原来的2倍)。可见如果不覆盖hashCode就不能正确的存储。

总结：get获取对象时，我们将K传给get，它调用hashCode计算hash从而得到Node位置，并进一步调用==或equals()方法确定键值对。可见为了正确的获取，要覆盖hashCode和equals方法

题外话：当链表长度超过8的时候，java8用红黑树代替了链表，目的是提高性能，这里不展开。HashMap是基于Map接口的实现，存储键值对时，它可以接收null的键值，是非同步的，HashMap存储着Entry(hash, key, value, next)对象。

Java中数据类型的可变和不可变

不可变数据类型： 当该数据类型的对应变量的值发生了改变，那么它对应的内存地址也会发生改变（真的是反向输出。。。），对于这种数据类型，就称不可变数据类型。其中基本数据类型都是不可变数据类型，例如int，如果一个int类型的数据发生改变，那么它指向了内存中的另一个地址，但是需要注意的是java缓存了所有-127-128的值。
可变数据类型 ：当该数据类型的对应变量的值发生了改变，那么它对应的内存地址不发生改变，对于这种数据类型，就称可变数据类型，当可变数据类型改变时它实际上是更改了内存中的内容。

举String的例子

String：不可变数据类型
StringBuilder：可变数据类型

不可变数据类型：对其修改会产生大量的临时拷贝(需要垃圾回收)
可变数据类型：最少化拷贝以提高效率，可以共享数据

可变数据类型风险在于想想集合吧，如果作为参数，它的值是会改变的

对于可变数据类型在非必须的情况下尽量不要重写这两个函数，使用原始的即可（比较内存是否相同），对于不可变数据类型，在重写equals时一定要重写hashCode否则若hashCode不相同时不会继续比较Equals

不可变类的特点：
  - 所有成员都是private final
  - 不提供对成员的改变方法，成员变量赋值一般在构造函数中赋值。
  - 确保所有的方法不会被重载：使用final定义class，或者类的所有方法加上final.
  - 如果某一个类成员不是原始变量或者不可变类，必须通过在成员初始化或者get方法时通过深度clone方法，来确保类的不可变。这个怎么说，看下面的例子
  ```java
  public final class ImmutableDemo {  
    private final int[] myArray;  
    public ImmutableDemo(int[] array) {  
        this.myArray = array; // wrong  
    }  
  }
  ```
  这种方式不能保证不可变性，myArray和array指向同一块内存地址，用户可以在ImmutableDemo之外通过修改array对象的值来改变myArray内部的值。为了保证内部的值不被修改，可以采用深度copy来创建一个新内存保存传入的值。正确做法：
  ```java
  public final class MyImmutableDemo {  
    private final int[] myArray;  
    public MyImmutableDemo(int[] array) {  
        this.myArray = array.clone();   
    }   
  }
  ```
  所以在getter方法中，不要直接返回对象本身，而是克隆对象，并返回对象的拷贝，这种做法也是防止对象外泄，防止通过getter获得内部可变成员对象后对成员变量直接操作，导致成员变量发生改变。

在来看上面提到的String对象的不可变性

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence
{
    /** The value is used for character storage. */
    private final char value[];
    /** The offset is the first index of the storage that is used. */
    private final int offset;
    /** The count is the number of characters in the String. */
    private final int count;
    /** Cache the hash code for the string */
    private int hash; // Default to 0
    ....
    public String(char value[]) {
         this.value = Arrays.copyOf(value, value.length); // deep copy操作
     }
    ...
     public char[] toCharArray() {
     // Cannot use Arrays.copyOf because of class initialization order issues
        char result[] = new char[value.length];
        System.arraycopy(value, 0, result, 0, value.length);
        return result;
    }
    ...
}
```

基本满足原则
1. String类被final修饰，不可继承
2. string内部所有成员都设置为私有变量
3. 不存在value的setter
4. 并将value和offset设置为final。
5. 当传入可变数组value[]时，进行copy而不是直接将value[]复制给内部变量.
6. 获取value时不是直接返回对象引用，而是返回对象的copy.

其实不可变的原因是
  - 不可变对象是线程安全的，在线程之间可以相互共享，不需要利用特殊机制来保证同步问题，因为对象的值无法改变。可以降低并发错误的可能性，因为不需要用一些锁机制等保证内存一致性问题也减少了同步开销
  - 易于构造、使用和测试

而String设计成不可变的话，优点在于

1. 字符串常量池的需要.
2. 线程安全考虑。
3. 类加载器要用到字符串，不可变性提供了安全性，以便正确的类被加载
4. 支持hash映射和缓存

虽然String对象将value设置为final,并且还通过各种机制保证其成员变量不可改变。但是还是可以通过反射机制的手段改变其值。例如

```java
//创建字符串"Hello World"， 并赋给引用s
String s = "Hello World";
System.out.println("s = " + s); //Hello World

//获取String类中的value字段
Field valueFieldOfString = String.class.getDeclaredField("value");
//改变value属性的访问权限
valueFieldOfString.setAccessible(true);

//获取s对象上的value属性的值
char[] value = (char[]) valueFieldOfString.get(s);
//改变value所引用的数组中的第5个字符
value[5] = '_';
System.out.println("s = " + s);  //Hello_World
```

#### 类和接口

1. 如果一个类实现了一个接口，那么接口中的所有的类方法在这个类中也都必须被声明为公有的，之所以如此，因为接口中的所有方法都隐含着公有访问级别，类实现去掉public修饰是不行的
2. 嵌套类有静态成员类，非静态成员类，匿名类，局部类，除了第一种，其他都被称为内部类，静态成员类是最普通的类，只是碰巧被声明在类的内部，它可以访问外围类的所有成员，包括那些被声明成私有的成员

补充一下匿名内部类的东西（Anonymous Classes）

匿名内部类可以使你的代码更加简洁，你可以在定义一个类的同时对其进行实例化。它与局部类很相似，不同的是它没有类名，如果某个局部类你只需要用一次，那么你就可以使用匿名内部类，匿名内部类本质上是一个重写或实现了父类或接口的子类对象。

匿名内部类使用要注意的地方：

1、定义匿名内部类
2、匿名内部类的语法
3、访问作用域的局部变量、定义和访问匿名内部类成员
4、匿名内部类实例

看个例子

```java
package com.qiming.test.anonymousclass;

public class HelloWorldAnonymousClasses {

  /**
   * 包含两个方法的HelloWorld接口
   */
  interface HelloWorld {
    public void greet();
    public void greetSomeone(String someone);
  }

  public void sayHello() {

    // 1、局部类EnglishGreeting实现了HelloWorld接口
    class EnglishGreeting implements HelloWorld {
      String name = "world";
      public void greet() {
        greetSomeone("world");
      }
      public void greetSomeone(String someone) {
        name = someone;
        System.out.println("Hello " + name);
      }
    }

    HelloWorld englishGreeting = new EnglishGreeting();

    // 2、匿名类实现HelloWorld接口
    HelloWorld frenchGreeting = new HelloWorld() {
      String name = "tout le monde";
      public void greet() {
        greetSomeone("tout le monde");
      }
      public void greetSomeone(String someone) {
        name = someone;
        System.out.println("Salut " + name);
      }
    };

    // 3、匿名类实现HelloWorld接口
    HelloWorld spanishGreeting = new HelloWorld() {
      String name = "mundo";
      public void greet() {
        greetSomeone("mundo");
      }
      public void greetSomeone(String someone) {
        name = someone;
        System.out.println("Hola, " + name);
      }
    };

    englishGreeting.greet();
    frenchGreeting.greetSomeone("Fred");
    spanishGreeting.greet();
  }

  public static void main(String... args) {
    HelloWorldAnonymousClasses myApp = new HelloWorldAnonymousClasses();
    myApp.sayHello();
  }

}

```

该例中用局部类来初始化变量englishGreeting，用匿类来初始化变量frenchGreeting和spanishGreeting，两种实现之间有明显的区别：

1、局部类EnglishGreetin继承HelloWorld接口，有自己的类名，定义完成之后需要再用new关键字实例化才可以使用；
2、frenchGreeting、spanishGreeting在定义的时候就实例化了，定义完了就可以直接使用；
3、匿名类是一个表达式，因此在定义的最后用分号";"结束。

匿名类是一个表达式，语法就类似于调用一个类的构造函数，除此之外，还包含一个代码块，

匿名内部类与局部类对作用域内的变量拥有相同的访问权限

1、匿名内部类可以访问外部类的所有成员
2、匿名内部类不能访问外部类类未加final修饰的变量（但是1.8中即使没有用final修饰也可以访问）
3、属性屏蔽，与内嵌类相同，匿名内部类定义的类型（如变量）会屏蔽其作用域范围内的其他同名类型（变量）

```java
package com.qiming.test.anonymousclass;

public class ShadowTest {

  public int x = 0;

  class FirstLevel {

    public int x = 1;

    void methodInFirstLevel(int x) {
      System.out.println("x = " + x);
      System.out.println("this.x = " + this.x);
      System.out.println("ShadowTest.this.x = " + ShadowTest.this.x);
    }
  }

  public static void main(String... args) {
    ShadowTest st = new ShadowTest();
    ShadowTest.FirstLevel fl = st.new FirstLevel();
    fl.methodInFirstLevel(23);
  }

}

```

输出结果是

x=23
this.x = 1
ShadowTest.this.x = 0

这个实例中有三个变量x：1、ShadowTest类的成员变量；2、内部类FirstLevel的成员变量；3、内部类方法methodInFirstLevel的参数。

4、匿名内部类中不能定义静态属性、方法
5、匿名内部类可以有常量属性
6、匿名内部类中可以定义属性
7、匿名内部类中可以有额外的方法（父接口，类中没有的方法）
8、匿名内部类中可以定义内部类
9、匿名内部类中可以对其他类进行实例化




#### 继承

1. 类是单继承的
2. **但是接口是多继承的**，因为看到了这样的代码
```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}
```
Executor框架中的内容，当时迟疑了一下，结果发现是真可以，如果多个接口中有相同方法，那么只有一个最近的方法被实现

#### 泛型

1. 泛型给了约束，让错误在编译时就能查出来，安全性和表述性上有优势，无限制通配符类型为Set<?>用一个?来表示，通配符类型是安全的，但是原生态类型则不安全(比如List)，因为泛型信息可以在运行时被擦除
2. @SuppressWarnings("unchecked")用此注解来解除告警，证明代码是类型安全的，自己保证

#### 枚举和注解

1. 注解是不支持继承的


#### 并发

1. ConcurrentHashMap采用了数组+Segment+分段锁的方式实现，定位一个元素需要两次Hash，第一次Hash定位到Segment，第二次Hash定位到元素所在链表的头部，ConcurrentHashMap理想情况下可以支持Segment数量大小的写操作，以上是在JDK1.7中，在JDK1.8中，参考了1.8的HashMap的实现，采用了数组+链表+红黑树的实现方式来设计，内部大量采用了CAS操作(比较交换)

2. CAS是一种乐观锁，悲观锁将资源锁住，等一个之前获得锁的线程释放锁后，下一个线程可以访问(Synchronized，lock)，而乐观锁采取一种宽泛的态度，通过某种方式不加锁来处理资源，比如给记录加version来获取数据，性能较悲观锁有很大的提高，CAS操作有三个操作数，内存位置(V)，预期位置(A)，新值(B)，如果内存地址里面的值和A的值是一样的，就将内存里面的值更新成B，CAS是通过无限循环来获取数据的。JDK8放弃了Segment，采用了Node

#### 知识点

**1、instanceof**

在重写equals方法时只有在要校验的目标类型是final的时候才可以使用instanceof进行类型判断，否则请使用getClass()。因为instanceof可以是所有超类或者超接口，而getClass不带继承关系


#### java中的util包

Collection中的集合，元素是孤立存在的（理解为单身），向集合中存储元素采用一个个元素的方式存储。
Map中的集合，元素是成对存在的(理解为夫妻)。每个元素由键与值两部分组成，通过键(K)可以找对所对应的值(V)。
Collection中的集合称为单列集合，Map中的集合称为双列集合。需要注意的是，Map中的集合不能包含重复的键，值可以重复；每个键只能对应一个值。

HashMap<K,V>：存储数据采用的哈希表结构，元素的存取顺序不能保证一致。由于要保证键的唯一、不重复，需要重写键的hashCode()方法、equals()方法。

LinkedHashMap<K,V>：HashMap下有个子类LinkedHashMap，存储数据采用的哈希表结构+链表结构。通过链表结构可以保证元素的存取顺序一致；通过哈希表结构可以保证的键的唯一、不重复，需要重写键的hashCode()方法、equals()方法。

常用的方法：

public V put(K key, V value): 把指定的键与指定的值添加到Map集合中。
public V remove(Object key): 把指定的键 所对应的键值对元素 在Map集合中删除，返回被删除元素的值。
public V get(Object key) 根据指定的键，在Map集合中获取对应的值。
boolean containsKey(Object key) 判断集合中是否包含指定的键。
public Set<K> keySet(): 获取Map集合中所有的键，存储到Set集合中。
public Set<Map.Entry<K,V>> entrySet(): 获取到Map集合中所有的键值对对象的集合(Set集合)。

Map的底层都是通过哈希表进行实现的，那先来看看什么是哈希表。

JDK1.8之前，哈希表底层采用数组+链表实现，即使用链表处理冲突，同一hash值的链表都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。

​JDK1.8中，哈希表存储采用数组+链表+红黑树实现，当链表长度超过阈值（8）时，将链表转换为红黑树，这样大大减少了查找时间。如下图（画的不好看，只是表达一个意思。

![hashmap]({{ "/assets/img/java/jdk8hashmap.png" | relative_url}})

说明

1，进行键值对存储时，先通过hashCode()计算出键（K）的哈希值，然后再数组中查询，如果没有则保存。

​2，但是如果找到相同的哈希值，那么接着调用equals方法判断它们的值是否相同。只有满足以上两种条件才能认定为相同的数据，因此对于Java中的包装类里面都重写了hashCode()和equals()方法。

​JDK1.8引入红黑树大程度优化了HashMap的性能，根据对象的hashCode和equals方法来决定的。如果我们往集合中存放自定义的对象，那么保证其唯一，就必须复写hashCode和equals方法建立属于当前对象的比较方式。

​总结：通过直接观看API文档中的解释，在结合哈希表的特点。我们得知为什么要使用hashCode()方法和equals()方法来作为元素是否相同的判断依据。

​1，使用hashCode()方法可以提高查询效率，假如现在有10个位置，存储某个元素如果说没有哈希值的使用，要查找该元素就要全部遍历，在效率上是缓慢的。而通过哈希值就可以很快定位到该元素的位置，节省了遍历数组的时间。

​2，但是通过哈希值就能确定唯一的值吗，当然不是。因此才需要使用equals再次进行判断。判断的目的在于当元素哈希值相等时，使用equals判断它们到底是不是同一个对象，如果是则代表是同一个元素，否则不是同一个元素那么就将其保存到链表上。

​因此哈希值的使用就是为提高查询速度，equals的使用就是判断对象是否为重复元素。

hashmap扩容是一件相对很耗时的事情，在初始化hash表结构时，如果没有指定大小则默认为16，也就是node数组的大小。当容量达到最大值时，扩容到原来的2倍。​ map集合的扩容要比list集合复杂的多。那么hashmap什么时候进行扩容呢？当hashmap中的元素个数超过数组大小*loadFactor时，就会进行数组扩容，loadFactor的默认值为0.75，也就是说，默认情况下，数组大小为16，那么当hashmap中元素个数超过16*0.75=12的时候，就把数组的大小扩展为2*16=32，即扩大一倍，然后重新计算每个元素在数组中的位置，而这是一个非常消耗性能的操作，所以如果我们已经预知hashmap中元素的个数，那么预设元素的个数能够有效的提高hashmap的性能。比如说，我们有1000个元素new HashMap(1000), 但是理论上来讲new HashMap(1024)更合适，不过上面annegu已经说过，即使是1000，hashmap也自动会将其设置为1024。 但是new HashMap(1024)还不是更合适的，因为0.75*1000 < 1000, 也就是说为了让0.75 * size > 1000, 我们必须这样new HashMap(2048)才最合适

对于LinkedHashMap而言，它继承于HashMap、底层使用哈希表与双向链表来保存所有元素。其基本操作与父类HashMap相似，它通过重写父类相关的方法，来实现自己的链接列表特性，同时它也保证元素是有序的。

Hashtable的实现原理和Hash Map是类似的，但区别是它是线程安全的，也正因为如此导致查询速度较慢。HashTable中K和V是不允许存储null值。

因为HashSet是另一个体系的（Set），但是底层就是通过HashMap实现的。HashSet的构造方法new了一个HashMap，最根本区别在于set集合存储单值对象。而map是键值对，但有一个相同点是都不能存储重复元素。 注意：Hash Set也是线程不安全的。

**补充一下红黑树**

红黑树是一种含有红黑结点并能自平衡的二叉查找树。它必须满足下面性质：

性质1：每个节点要么是黑色，要么是红色。
性质2：根节点是黑色。
性质3：每个叶子节点（NIL）是黑色。
性质4：每个红色结点的两个子结点一定都是黑色。
性质5：任意一结点到每个叶子结点的路径都包含数量相同的黑结点。
从性质5又可以推出：
性质5.1：如果一个结点存在黑子结点，那么该结点肯定有两个子结点

红黑树总是通过旋转和变色达到自平衡，红黑树引入了“颜色”的概念。引入“颜色”的目的在于使得红黑树的平衡条件得以简化，红黑树并不追求“完全平衡”——它只要求部分地达到平衡要求，降低了对旋转的要求，从而提高了性能。红黑树能够以O(log2 n)的时间复杂度进行搜索、插入、删除操作。此外，由于它的设计，任何不平衡都会在三次旋转之内解决。当然，还有一些更好的，但实现起来更复杂的数据结构能够做到一步旋转之内达到平衡，但红黑树能够给我们一个比较“便宜”的解决方案。红黑树的算法时间复杂度和AVL相同，但统计性能比AVL树更高

当然，红黑树并不适应所有应用树的领域。如果数据基本上是静态的，那么让他们待在他们能够插入，并且不影响平衡的地方会具有更好的性能。如果数据完全是静态的，例如，做一个哈希表，性能可能会更好一些。  

在实际的系统中，例如，需要使用动态规则的防火墙系统，使用红黑树而不是散列表被实践证明具有更好的伸缩性。

参考[文章](jianshu.com/p/e136ec79235c)

所以整个的总结可以如下

List：ArrayList、LinkedList、Vector及Stack
  - ArrayList，可以重复值，有序（指插入顺序），基于数组实现，无容量限制，非线程安全，ArrayList找一个元素就是遍历数组（contains/indexOf）
  - LinkedList，可以重复值，有序（指插入顺序），基于双向链表实现，非线程安全，LinkedList找一个元素就是遍历双端链表（contains/indexOf）
  - Vector，可以重复值，有序，基于ArrayList但是线程安全的
  - Stack，可以重复值，栈操作，继承自Vector
Set：HashSet、TreeSet
  - HashSet，不可以重复值，无序，默认构造一个HashMap对象，非线程安全，HashSet里面找一个元素其实是去hashmap里面找一个key（因为add的时候就是将对象作为了一个key，value好像是给了个固定值，所以contains比较的时候不需要使用value）
  - TreeSet，不可以重复值，有序（指按数值排序），基于TreeMap实现，非线程安全

Queue：ArrayDeque、LinkedList、PriorityQueue、ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue、ConcurrentLinkedQueue、DelayQueue
  - ArrayDeque，双端队列，数组实现
  - LinkedList，对，没错是LinkedList，双端队列，链表实现
  - PriorityQueue，通过堆实现，有也就是完全二叉树实现的小顶堆，这个完全二叉树是用数组存储的
  - ArrayBlockingQueue，看名字，数组实现的阻塞队列，线程安全的。ArrayBlockingQueue初始化时必须传入容量，还有一个参数就是公平与否的锁，它的长度是固定的，如果消费速度跟不上入队速度，则会导致提供者线程一直阻塞，非常危险，用的是一个ReentrantLock来控制入队和出队的
  - ConcurrentLinkedQueue，非阻塞的实现线程安全（阻塞的就是使用一个或两个ReentrantLock），而非阻塞的是使用循环的CAS，看名字，特性是无界，非双端队列，循环CAS入队尾结点。head和tail并非总是指向队列的头/尾结点，也就是说允许队列处于不一致状态，使用三个不变式来维护非阻塞算法的正确性。
  - LinkedBlockingQueue，线程安全，这是一个只能一端出，一端入的单向队列结构，特点是无界，且有两把锁ReentrantLock，一个是take锁，一个是put锁，那队尾添加的时候，队头仍然可以进行删除操作，remove()操作就是要同时获得两把锁
  - LinkedBlockingDeque，很明白了吧，两端可以同时出入，双端队列（也就是有prev），但是它是一个ReentrantLock。
  - DelayQueue，无界的BlockingQueue，对象只能在其到期时才能从队列中取走，这种队列是有序的，即队头对象的延迟到期时间最长，null元素不要放到这种队列中。有很多场景适用。只能添加实现Delayed接口的对象。构造函数一般都有一个时间传入，只有剩余时间为0的时候，该元素才有资格从队列中取出，且这些元素是排序的
  - PriorityBlockingQueue，特殊的无界优先级阻塞队列，跟PriorityQueue一样，二叉树是用数组存储的，线程安全的。内部使用堆算法保证每次出队都是优先级最高（或者最低）的元素，allocationSpinLock 是个自旋锁，用CAS操作来保证只有一个线程可以扩容队列，状态为0 或者1，其中0标示当前没有在进行扩容，1标示当前正在扩容。默认队列容量是11，offer()操作，由于是无界队列，所以一直返回true，

非Collection的Map接口

Map：HashMap、TreeMap、LinkedHashMap、HashTable、ConcurrentHashMap
  - HashMap，键不能重复，值可以，无序，基于数组、链表、红黑树，无容量限制，非线程安全
  - HashTable，不可以存放null值，线程安全
  - LinkedHashMap，键不能重复，值可以，有序，基于HashMap，非线程安全
  - TreeMap，TreeMap基于红黑树的实现，因此它要求一定要有key比较的方法，要么传入Comparator实现，要么key对象实现Comparable借口。在put操作时，基于红黑树的方式遍历，基于comparator来比较key应放在树的左边还是右边，如找到相等的key，则直接替换掉value，有序（指按数值排序）
  - ConcurrentHashMap，解决并发问题的HashMap，线程安全

#### java中equals和hashcode

equals()：反映的是对象或变量具体的值，可能是对象的引用，也可能是值类型的值。

hashCode()：计算出对象实例的哈希码，并返回哈希码，又称为散列函数。根类Object的hashCode()方法是native方法，实现逻辑与JVM有关，有些JVM是直接返回对象的存储地址，但是大多时候并不是这样，只能说可能和存储地址有一定关联，每个Object对象的hashCode都是唯一的；当然，当对象所对应的类重写了hashCode()方法时，结果就截然不同了。

之所以有hashCode方法，是因为在批量的对象比较中，hashCode要比equals来得快，很多集合都用到了hashCode，比如HashTable。

```html

两个obj，如果equals()相等，hashCode()一定相等。equals()相等的两个对象，hashcode()一定相等。

两个obj，如果hashCode()相等，equals()不一定相等（Hash散列值有冲突的情况，虽然概率很低）。hashcode()不等，一定能推出equals()也不等。


```

注意，判断两个对象是否相等的规则是：

第一步，如果hashCode()相等，则查看第二步，否则不相等;

第二步，查看equals()是否相等，如果相等，则两obj相等，否则还是不相等。

为什么要重写equals方法？

我们先看看Object中equals方法的源码：

public boolean equals(Object obj) { 

         return (this == obj); 

}

“==”是比较两个对象的的内存地址，所以说使用Object的equals()方法是比较两个对象的内存地址是否相等。因为Object的equal方法默认是两个对象的引用的比较，如果现在需要利用对象里面的值来判断是否相等，则需要重载equal方法。String、Double、Integer、Math这些类对equals方法改写，进行的是内容的比较。

```html

equals重写约定

自反性:  x.equals(x) 一定是true。

对null:  x.equals(null) 一定是false。

对称性:  x.equals(y)和y.equals(x)结果一致。

传递性：x.equals(y), 并且y.equals(z)，那么x.equals(z)。

一致性：对于任何非null的引用值x和y，只要equals的比较操作在对象中所用的信息没有被修改，多次调用x.equals(y)返回结果一致;因此，equals方法里面不应该依赖任何不可靠的资源。

hashCode重写约定

在某个运行时期间，只要对象的变化不会影响equals方法的决策结果，那么，在这个期间，无论调用多少次hashCode，都必须返回同一个散列码。

通过equals调用返回true的2个对象的hashCode一定一样。

通过equasl返回false的2个对象的散列码不需要不同，也就是他们的hashCode方法的返回值允许出现相同的情况。

总结一句话：等价的(调用equals返回true)对象必须产生相同的散列码。不等价的对象，不要求产生的散列码不相同。

为了更好地遵从上面的约定，可以如此规范：重写了euqls方法的对象必须同时重写hashCode方法。

```

改写equals时总是要改写hashcode

比如改写一个类的equals方法，让它去比较内容而不是引用是否相同，又不去改写hashcode方法，那么就会出现一个问题，就是我new两个内容一样的对象，但是它们的地址是不同的，equals方法得到true，但是hashcode方法依赖与地址可能就会得到false，这显然不行，所以我们必须改写hashcode方法使他根据内容判断（equals相同，hashcode必须相同）。

#### java中设计一个可以用作Map的Key的类

```java

public class Person {
  private String id;

  public Person(String id) {
    this.id = id
  }
}

public static void main(String[] args) {

  HashMap<Person, String> map = new HashMap<Person, String>();
  map.put(new Person("001"), "111");
  map.put(new Person("002"), "222");
  map.put(new Person("003"), "333");
  map.put(new Person("003"), "3333");

  System.out.println(map.toString);

  System.out.println(map.get(new Person("001")));
  System.out.println(map.get(new Person("002")));
  System.out.println(map.get(new Person("003")));
}

```

上面这个肯定是不行的，没有重写equals和hashcode

在HashMap中，查找key的比较顺序为：

1、计算对象的Hash Code，看在表中是否存在。
2、检查对应Hash Code位置中的对象和当前对象是否相等。

显然，第一步就是要用到hashCode()方法，而第二步就是要用到equals()方法。在没有进行重载时，在这两步会默认调用Object类的这两个方法，而在Object中，Hash Code的计算方法是根据对象的地址进行计算的，那两个Person("003")的对象地址是不同的，所以它们的Hash Code也不同，自然HashMap也不会把它们当成是同一个key了。同时，在Object默认的equals()中，也是根据对象的地址进行比较，自然一个Person("003")和另一个Person("003")是不相等的。

到这里，有点感觉，那么HashMap的Key可以是一个可变对象吗？可变对象是指创建后自身状态能改变的对象。换句话说，可变对象是该对象在创建后它的哈希值（由类的hashCode（）方法可以得出哈希值）可能被改变。（更具体的看上面的第5点，说的有点绕的，这边就简单点）

```java

public class MutableKey {
    private int i;
    private int j;
 
    public MutableKey(int i, int j) {
        this.i = i;
        this.j = j;
    }
 
    public final int getI() {
        return i;
    }
 
    public final void setI(int i) {
        this.i = i;
    }
 
    public final int getJ() {
        return j;
    }
 
    public final void setJ(int j) {
        this.j = j;
    }
 
    @Override
    public int hashCode() {
        final int prime = 31;
        int result = 1;
        result = prime * result + i;
        result = prime * result + j;
        return result;
    }
 
    @Override
    public boolean equals(Object obj) {
        if (this == obj) {
            return true;
        }
        if (obj == null) {
            return false;
        }
        if (!(obj instanceof MutableKey)) {
            return false;
        }
        MutableKey other = (MutableKey) obj;
        if (i != other.i) {
            return false;
        }
        if (j != other.j) {
            return false;
        }
        return true;
    }
}

public class MutableDemo {

    public static void main(String[] args) {

        // Object created
        MutableKey key = new MutableKey(10, 20);
        System.out.println("Hash code: " + key.hashCode());

        // Object State is changed after object creation.
        key.setI(30);
        key.setJ(40);
        System.out.println("Hash code: " + key.hashCode());
    }
}

```


在下面的代码中，对象MutableKey的键在创建时变量 i=10 j=20，哈希值是1291。

然后我们改变实例的变量值，该对象的键 i 和 j 从10和20分别改变成30和40。现在Key的哈希值已经变成1931。

显然，这个对象的键在创建后发生了改变。所以类MutableKey是可变的。

HashMap底层是使用Entry对象数组存储的，而Entry是一个单项的链表。当调用一个put（）方法将一个键值对添加进来是，先使用hash（）函数获取该对象的hash值，然后调用indexFor方法查找到该对象在数组中应该存储的下标，假如该位置为空，就将value值插入，如果该下标出不为空，则要遍历该下标上面的对象，使用equals方法进行判断，如果遇到equals（）方法返回真的则进行替换，否则将其插入，源码详解可看：

在HashMap中使用可变对象作为Key带来的问题！！

```java

public V get(Object key)   
{   
 // 如果 key 是 null，调用 getForNullKey 取出对应的 value   
 if (key == null)   
     return getForNullKey();   
 // 根据该 key 的 hashCode 值计算它的 hash 码  
 int hash = hash(key.hashCode());   
 // 直接取出 table 数组中指定索引处的值，  
 for (Entry<K,V> e = table[indexFor(hash, table.length)];   
     e != null;   
     // 搜索该 Entry 链的下一个 Entr   
     e = e.next)         // ①  
 {   
     Object k;   
     // 如果该 Entry 的 key 与被搜索 key 相同  
     if (e.hash == hash && ((k = e.key) == key   
         || key.equals(k)))   
         return e.value;   
 }   
 return null;   
}

```

上面是HashMap的get（）方法源码，通过上面我们可以知道，如果 HashMap 的每个 bucket 里只有一个 Entry 时，HashMap 可以根据索引、快速地取出该 bucket 里的 Entry；在发生“Hash 冲突”的情况下，单个 bucket 里存储的不是一个 Entry，而是一个 Entry 链，系统只能必须按顺序遍历每个 Entry，直到找到想搜索的 Entry 为止——如果恰好要搜索的 Entry 位于该 Entry 链的最末端（该 Entry 是最早放入该 bucket 中），那系统必须循环到最后才能找到该元素。

同时我们也看到，判断是否找到该对象，我们还需要判断他的哈希值是否相同，假如哈希值不相同，根本就找不到我们要找的值。

如果Key对象是可变的，那么Key的哈希值就可能改变。在HashMap中可变对象作为Key会造成数据丢失。

我们也能定义属于自己的不可变类。

如果可变对象在HashMap中被用作键，那就要小心在改变对象状态的时候，不要改变它的哈希值了。我们只需要保证成员变量的改变能保证该对象的哈希值不变即可。

在下面的Employee示例类中，哈希值是用实例变量id来计算的。一旦Employee的对象被创建，id的值就不能再改变。只有name可以改变，但name不能用来计算哈希值。所以，一旦Employee对象被创建，它的哈希值不会改变。所以Employee在HashMap中用作Key是安全的。

```java

import java.util.HashMap;
import java.util.Map;

public class MutableSafeKeyDemo {

    public static void main(String[] args) {
        Employee emp = new Employee(2);
        emp.setName("Robin");

        // Put object in HashMap.
        Map<Employee, String> map = new HashMap<>();
        map.put(emp, "Showbasky");

        System.out.println(map.get(emp));

        // Change Employee name. Change in 'name' has no effect
        // on hash code.
        emp.setName("Lily");
        System.out.println(map.get(emp));
    }
}

class Employee {
    // It is specified while object creation.
    // Cannot be changed once object is created. No setter for this field.
    private int id;
    private String name;

    public Employee(final int id) {
        this.id = id;
    }

    public final String getName() {
        return name;
    }

    public final void setName(final String name) {
        this.name = name;
    }

    public int getId() {
        return id;
    }

    // Hash code depends only on 'id' which cannot be
    // changed once object is created. So hash code will not change
    // on object's state change
    @Override
    public int hashCode() {
        final int prime = 31;
        int result = 1;
        result = prime * result + id;
        return result;
    }

    @Override
    public boolean equals(Object obj) {
        if (this == obj)
            return true;
        if (obj == null)
            return false;
        if (getClass() != obj.getClass())
            return false;
        Employee other = (Employee) obj;
        if (id != other.id)
            return false;
        return true;
    }
}

```

#### java中的hashmap再详解

(1) HashMap：它根据键的hashCode值存储数据，大多数情况下可以直接定位到它的值，因而具有很快的访问速度，但遍历顺序却是不确定的。 HashMap最多只允许一条记录的键为null，允许多条记录的值为null。HashMap非线程安全，即任一时刻可以有多个线程同时写HashMap，可能会导致数据的不一致。如果需要满足线程安全，可以用 Collections的synchronizedMap方法使HashMap具有线程安全的能力，或者使用ConcurrentHashMap。

(2) Hashtable：Hashtable是遗留类，很多映射的常用功能与HashMap类似，不同的是它承自Dictionary类，并且是线程安全的，任一时间只有一个线程能写Hashtable，并发性不如ConcurrentHashMap，因为ConcurrentHashMap引入了分段锁。Hashtable不建议在新代码中使用，不需要线程安全的场合可以用HashMap替换，需要线程安全的场合可以用ConcurrentHashMap替换。

(3) LinkedHashMap：LinkedHashMap是HashMap的一个子类，保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的，也可以在构造时带参数，按照访问次序排序。

(4) TreeMap：TreeMap实现SortedMap接口，能够把它保存的记录根据键排序，默认是按键值的升序排序，也可以指定排序的比较器，当用Iterator遍历TreeMap时，得到的记录是排过序的。如果使用排序的映射，建议使用TreeMap。在使用TreeMap时，key必须实现Comparable接口或者在构造TreeMap传入自定义的Comparator，否则会在运行时抛出java.lang.ClassCastException类型的异常。

对于上述四种Map类型的类，要求映射中的key是不可变对象。不可变对象是该对象在创建后它的哈希值不会被改变。如果对象的哈希值发生变化，Map对象很可能就定位不到映射的位置了。

通过上面的比较，我们知道了HashMap是Java的Map家族中一个普通成员，鉴于它可以满足大多数场景的使用条件，所以是使用频度最高的一个。下文我们主要结合源码，从存储结构、常用方法分析、扩容以及安全性等方面深入讲解HashMap的工作原理。

存储结构-字段
从结构实现来讲，HashMap是数组+链表+红黑树（JDK1.8增加了红黑树部分）实现的，如下如所示。

(1) 从源码可知，HashMap类中有一个非常重要的字段，就是 Node[] table，即哈希桶数组，明显它是一个Node的数组。我们来看Node[JDK1.8]是何物。

```java

static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;    //用来定位数组索引位置
        final K key;
        V value;
        Node<K,V> next;   //链表的下一个node

        Node(int hash, K key, V value, Node<K,V> next) { ... }
        public final K getKey(){ ... }
        public final V getValue() { ... }
        public final String toString() { ... }
        public final int hashCode() { ... }
        public final V setValue(V newValue) { ... }
        public final boolean equals(Object o) { ... }
}

```

(2) HashMap就是使用哈希表来存储的。哈希表为解决冲突，可以采用开放地址法和链地址法等来解决问题，Java中HashMap采用了链地址法。链地址法，简单来说，就是数组加链表的结合。在每个数组元素上都一个链表结构，

```java

map.put("明野","小明");

```

系统将调用”明野”这个key的hashCode()方法得到其hashCode 值（该方法适用于每个Java对象），然后再通过Hash算法的后两步运算（高位运算和取模运算，下文有介绍）来定位该键值对的存储位置，有时两个key会定位到相同的位置，表示发生了Hash碰撞。当然Hash算法计算结果越分散均匀，Hash碰撞的概率就越小，map的存取效率就会越高。

如果哈希桶数组很大，即使较差的Hash算法也会比较分散，如果哈希桶数组数组很小，即使好的Hash算法也会出现较多碰撞，所以就需要在空间成本和时间成本之间权衡，其实就是在根据实际情况确定哈希桶数组的大小，并在此基础上设计好的hash算法减少Hash碰撞。那么通过什么方式来控制map使得Hash碰撞的概率又小，哈希桶数组（Node[] table）占用空间又少呢？答案就是好的Hash算法和扩容机制。

在理解Hash和扩容流程之前，我们得先了解下HashMap的几个字段。从HashMap的默认构造函数源码可知，构造函数就是对下面几个字段进行初始化，源码如下：

```java

int threshold;             // 所能容纳的key-value对极限
final float loadFactor;    // 负载因子
int modCount;  
int size;  

```

首先，Node[] table的初始化长度length(默认值是16)，Load factor为负载因子(默认值是0.75)，threshold是HashMap所能容纳的最大数据量的Node(键值对)个数。threshold = length * Load factor。也就是说，在数组定义好长度之后，负载因子越大，所能容纳的键值对个数越多。

结合负载因子的定义公式可知，threshold就是在此Load factor和length(数组长度)对应下允许的最大元素数目，超过这个数目就重新resize(扩容)，扩容后的HashMap容量是之前容量的两倍。默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议大家不要修改，除非在时间和空间比较特殊的情况下，如果内存空间很多而又对时间效率要求很高，可以降低负载因子Load factor的值；相反，如果内存空间紧张而对时间效率要求不高，可以增加负载因子loadFactor的值，这个值可以大于1。

size这个字段其实很好理解，就是HashMap中实际存在的键值对数量。注意和table的长度length、容纳最大键值对数量threshold的区别。而modCount字段主要用来记录HashMap内部结构发生变化的次数，主要用于迭代的快速失败。强调一点，内部结构发生变化指的是结构发生变化，例如put新键值对，但是某个key对应的value值被覆盖不属于结构变化。

在HashMap中，哈希桶数组table的长度length大小必须为2的n次方(一定是合数)，这是一种非常规的设计，常规的设计是把桶的大小设计为素数。相对来说素数导致冲突的概率要小于合数，具体证明可以参考这篇文章，Hashtable初始化桶大小为11，就是桶大小设计为素数的应用（Hashtable扩容后不能保证还是素数）。HashMap采用这种非常规设计，主要是为了在取模和扩容时做优化，同时为了减少冲突，HashMap定位哈希桶索引位置时，也加入了高位参与运算的过程。

这里存在一个问题，即使负载因子和Hash算法设计的再合理，也免不了会出现拉链过长的情况，一旦出现拉链过长，则会严重影响HashMap的性能。于是，在JDK1.8版本中，对数据结构做了进一步的优化，引入了红黑树。而当链表长度太长（默认超过8）时，链表就转换为红黑树，利用红黑树快速增删改查的特点提高HashMap的性能，其中会用到红黑树的插入、删除、查找等算法。本文不再对红黑树展开讨论，想了解更多红黑树数据结构的工作原理可以参考

HashMap的内部功能实现很多，本文主要从根据key获取哈希桶数组索引位置、put方法的详细执行、扩容过程三个具有代表性的点深入展开讲解。

1. 确定哈希桶数组索引位置
不管增加、删除、查找键值对，定位到哈希桶数组的位置都是很关键的第一步。前面说过HashMap的数据结构是数组和链表的结合，所以我们当然希望这个HashMap里面的元素位置尽量分布均匀些，尽量使得每个位置上的元素数量只有一个，那么当我们用hash算法求得这个位置的时候，马上就可以知道对应位置的元素就是我们要的，不用遍历链表，大大优化了查询的效率。HashMap定位数组索引位置，直接决定了hash方法的离散性能。先看看源码的实现(方法一+方法二):

```java

方法一：
static final int hash(Object key) {   //jdk1.8 & jdk1.7
     int h;
     // h = key.hashCode() 为第一步 取hashCode值
     // h ^ (h >>> 16)  为第二步 高位参与运算
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
方法二：
static int indexFor(int h, int length) {  //jdk1.7的源码，jdk1.8没有这个方法，但是实现原理一样的
     return h & (length-1);  //第三步 取模运算
}

// 1.8的代码，看first = tab[(n - 1) & hash] 和 h & (length-1) 是一个意思的定位

final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```

这里的Hash算法本质上就是三步：取key的hashCode值、高位运算、取模运算。

对于任意给定的对象，只要它的hashCode()返回值相同，那么程序调用方法一所计算得到的Hash码值总是相同的。我们首先想到的就是把hash值对数组长度取模运算，这样一来，元素的分布相对来说是比较均匀的。但是，模运算的消耗还是比较大的，在HashMap中是这样做的：调用方法二来计算该对象应该保存在table数组的哪个索引处。

这个方法非常巧妙，它通过h & (table.length -1)来得到该对象的保存位，而HashMap底层数组的长度总是2的n次方，这是HashMap在速度上的优化。当length总是2的n次方时，h& (length-1)运算等价于对length取模，也就是h%length，但是&比%具有更高的效率。

在JDK1.8的实现中，优化了高位运算的算法，通过hashCode()的高16位异或低16位实现的：(h = k.hashCode()) ^ (h >>> 16)，主要是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

[原文](https://tech.meituan.com/2016/06/24/java-hashmap.html)

```html

// 调用hashcode()
1111111111111111 1111000011101010  

// 计算hash

h >>> 16

0000000000000000 1111111111111111

hash = h ^ (h >>> 16) // ^是异或

1111111111111111 0000111100010101

(n - 1) & hash // n 默认是16， -1 就是15

0000000000000000 0000000000001111
1111111111111111 0000111100010101

// 结果

0101 = 5

```

分析下HashMap的put方法

1、判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；
2、根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向6，如果table[i]不为空，转向3
3、判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向4，这里的相同指的是hashCode以及equals；
4、判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向5
5、遍历table[i]，判断链表长度是否大于8，大于8的话把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可
6、插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容

```java

1 public V put(K key, V value) {
 2     // 对key的hashCode()做hash
 3     return putVal(hash(key), key, value, false, true);
 4 }
 5
 6 final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
 7                boolean evict) {
 8     Node<K,V>[] tab; Node<K,V> p; int n, i;
 9     // 步骤1：tab为空则创建
10     if ((tab = table) == null || (n = tab.length) == 0)
11         n = (tab = resize()).length;
12     // 步骤2：计算index，并对null做处理
13     if ((p = tab[i = (n - 1) & hash]) == null)
14         tab[i] = newNode(hash, key, value, null);
15     else {
16         Node<K,V> e; K k;
17         // 步骤3：节点key存在，直接覆盖value
18         if (p.hash == hash &&
19             ((k = p.key) == key || (key != null && key.equals(k))))
20             e = p;
21         // 步骤4：判断该链为红黑树
22         else if (p instanceof TreeNode)
23             e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
24         // 步骤5：该链为链表
25         else {
26             for (int binCount = 0; ; ++binCount) {
27                 if ((e = p.next) == null) {
28                     p.next = newNode(hash, key,value,null);
                        //链表长度大于8转换为红黑树进行处理
29                     if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st  
30                         treeifyBin(tab, hash);
31                     break;
32                 }
                    // key已经存在直接覆盖value
33                 if (e.hash == hash &&
34                     ((k = e.key) == key || (key != null && key.equals(k))))
35							break;
36                 p = e;
37             }
38         }
39         
40         if (e != null) { // existing mapping for key
41             V oldValue = e.value;
42             if (!onlyIfAbsent || oldValue == null)
43                 e.value = value;
44             afterNodeAccess(e);
45             return oldValue;
46         }
47     }

48     ++modCount;
49     // 步骤6：超过最大容量 就扩容
50     if (++size > threshold)
51         resize();
52     afterNodeInsertion(evict);
53     return null;
54 }

```

扩容机制


扩容(resize)就是重新计算容量，向HashMap对象里不停的添加元素，而HashMap对象内部的数组无法装载更多的元素时，对象就需要扩大数组的长度，以便能装入更多的元素。当然Java里的数组是无法自动扩容的，方法是使用一个新的数组代替已有的容量小的数组，就像我们用一个小桶装水，如果想装更多的水，就得换大水桶。

我们分析下resize的源码，鉴于JDK1.8融入了红黑树，较复杂，为了便于理解我们仍然使用JDK1.7的代码，好理解一些，本质上区别不大，具体区别后文再说。

```java

1 void resize(int newCapacity) {   //传入新的容量
2     Entry[] oldTable = table;    //引用扩容前的Entry数组
3     int oldCapacity = oldTable.length;         
4     if (oldCapacity == MAXIMUM_CAPACITY) {  //扩容前的数组大小如果已经达到最大(2^30)了
5         threshold = Integer.MAX_VALUE; //修改阈值为int的最大值(2^31-1)，这样以后就不会扩容了
6         return;
7     }
8  
9     Entry[] newTable = new Entry[newCapacity];  //初始化一个新的Entry数组
10     transfer(newTable);                         //！！将数据转移到新的Entry数组里
11     table = newTable;                           //HashMap的table属性引用新的Entry数组
12     threshold = (int)(newCapacity * loadFactor);//修改阈值
13 }

```

这里就是使用一个容量更大的数组来代替已有的容量小的数组，transfer()方法将原有Entry数组的元素拷贝到新的Entry数组里。

```java

1 void transfer(Entry[] newTable) {
2     Entry[] src = table;                   //src引用了旧的Entry数组
3     int newCapacity = newTable.length;
4     for (int j = 0; j < src.length; j++) { //遍历旧的Entry数组
5         Entry<K,V> e = src[j];             //取得旧Entry数组的每个元素
6         if (e != null) {
7             src[j] = null;//释放旧Entry数组的对象引用（for循环后，旧的Entry数组不再引用任何对象）
8             do {
9                 Entry<K,V> next = e.next;
10                 int i = indexFor(e.hash, newCapacity); //！！重新计算每个元素在数组中的位置
11                 e.next = newTable[i]; //标记[1]
12                 newTable[i] = e;      //将元素放在数组上
13                 e = next;             //访问下一个Entry链上的元素
14             } while (e != null);
15         }
16     }
17 }

```

newTable[i]的引用赋给了e.next，也就是使用了单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置；这样先放在一个索引上的元素终会被放到Entry链的尾部(如果发生了hash冲突的话），这一点和Jdk1.8有区别，下文详解。在旧数组中同一条Entry链上的元素，通过重新计算索引位置后，有可能被放到了新数组的不同位置上。


#### Java中一些基本技巧

1、取余(%)和取模(Math.floorMod)

不要觉得取余和取模式一样的。。只是我们用的基本都是两个数字的符号一直
```java
Math.floorMod(+4, -3) == -2;    (+4 % -3) == +1;
Math.floorMod(-4, +3) == +2;    (-4 % +3) == -1;
Math.floorMod(-4, -3) == -1;    (-4 % -3) == -1;
Math.floorMod(+4, +3) == +1;    (+4 % +3) == +1;
```
当y≠0时：

取余：rem(x,y)=x-y.*fix(x./y)

取模：mod(x,y)=x-y.*floor(x./y)
其中，fix()函数是向0取整，floor()函数是向负无穷取整

例如： 4 / (-3) 约等于-1.3
在取余运算时候商值向0方向舍弃小数位于是fix(-1.3) = -1
取余结果 : 4 - (-3)(-1) = 1
在取模运算时商值向负无穷方向舍弃小数位于是 floor(-1.3) = -2
取模结果 : 4 - (-3)(-2) = -2

取余的时候符号和被除数保持一致，取模的时候符号和除数保持一致。

2、集合中的一些用法总结

这边要提到的是JDK8中集合增加的一些API

Map接口中有的方法：

Map.getOrDefault(Object, V)，许调用者在代码语句中规定获得在map中符合提供的键的值，否则在没有找到提供的键的匹配项的时候返回一个“默认值”。

Map.putIfAbsent(K,V)，在缺失键的时候才put，不像以前要判断，一行代码

Map.remove(Object.Object)，只有在提供的键和值都匹配的时候才会删除该map项（之前的有效版本只是查找“键”的匹配来删除）。不用再写值的判断比较

Map.replace(K,V)，只有在指定的键已经存在并且有与之相关的映射值时才会将指定的键映射到指定的值（新值）

Map.replace(K,V,V)，只是涵盖指定的键在映射中有任意一个有效的值的替换处理，而这个“replace”方法接受一个额外的（第三个）参数，只有在指定的键和值都匹配的情况下才会替换。中间是旧值，最后一个是新值

3、Collections类

Collections类不是Collection接口哦，工具类Collectons用于操作集合类，如list，set。提供的所有方法都是静态的（注意没有Map接口的哦，但是Map的values出来的是Collection接口的，就可以用Collections类操作了）

主要的有：

a.sort方法的使用

对集合进行排序,默认按照升序排序，列表中所有元素必须实现Comparable接口

b.reverse方法的使用

对集合中元素进行反转

c.shuffle方法的使用

对集合中元素进行随机排序

d.fill(list list,Object o)方法使用

用对象o替换list中的所有元素

e.copy（List m，List n）方法的使用

将集合n中的元素全部复制到m中，并且覆盖相应索引的元素，也就是可以不等长

f.min/max（Collection）,min/max（Collection,Comparator）方法的使用

前者采用Collection的自然比较法，后者采用Comparator进行比较

g.indexOfSubList（List m,List n）方法的使用

查找n在m中首次出现位置的索引

h.lastIndexOfSubList（List m,List n）方法的使用

查找n在m中最后出现位置的索引

i.rotate(List list,int m)方法的使用

集合中的元素向后移动m个位置，在后面被覆盖的元素循环到前面。m是负数表示向左移动，m是正数表示向右移动

j.swap(List list,int i,int j)方法的使用

交换集合中指定元素索引的元素

k.binarySearch(Collection,Object)方法的使用

查找指定集合中的元素，返回所查找元素位置的索引，看binarySearch就知道是二分查找搞的，这个二分中不带排序，所以使用前要保证Collection有序

l.replaceAll(List list,Object old,Object new)方法的使用

替换指定元素为新元素，若被替换的元素存在返回true，反之返回false


4、一些基本类型的总结

把char字符型数字转成int数字，因为他们的ascii码值恰好相差48， 因此把char型数字减去48得到int型数据，例如'4'转换成了4。而'0'的ascii码是48

ASCII码占用一个字节，可以有0～255共256个取值。前128个为常用的字符如运算符，字母 ，数字等 键盘上可以显示的后 128个为 特殊字符是键盘上找不到的字符。还有'A'是65，'Z'是106。而'a'是113，'z'是122。大小写是不连续的


5、Java的值传递和引用传递

[看这篇](https://segmentfault.com/a/1190000016773324)

形参：方法被调用时需要传递进来的参数，如：func(int a)中的a，它只有在func被调用期间a才有意义，也就是会被分配内存空间，在方法func执行完成后，a就会被销毁释放空间，也就是不存在了

实参：方法被调用时是传入的实际值，它在方法被调用前就已经被初始化并且在方法被调用时传入。

```java

public static void func(int a){
 a=20;
 System.out.println(a);
}
public static void main(String[] args) {
 int a=10;//实参
 func(a);
}

```
int a=10;中的a在被调用之前就已经创建并初始化，在调用func方法时，他被当做参数传入，所以这个a是实参。

而func(int a)中的a只有在func被调用时它的生命周期才开始，而在func调用结束之后，它也随之被JVM释放掉，，所以这个a是形参。

A.基本数据类型的局部变量（方法中的）

第一点：我们声明并初始化基本数据类型的局部变量时，变量名以及字面量值都是存储在栈中，而且是真实的内容。栈中的数据在当前线程下是共享的

第二点：基本数据类型的数据本身是不会改变的，当局部变量重新赋值时，并不是在内存中改变字面量内容，而是重新在栈中寻找已存在的相同的数据，若栈中不存在，则重新开辟内存存新数据，并且把要重新赋值的局部变量的引用指向新数据所在地址。

B. 基本数据类型的成员变量（成员变量：顾名思义，就是在类体中定义的变量。）

```java
public class Person{
  private int age;
  private String name;
  private int grade;
//篇幅较长，省略setter getter方法
  static void run(){
     System.out.println("run....");
   };
}

//调用
Person per = new Person();
```

per的地址指向的是堆内存中的一块区域 栈，per -> 【x0123】 ->  | 堆，int age; String name; int grade; static void run();

第一点：基本数据类型的成员变量名和值都存储于堆中，其生命周期和对象的是一致的。

C. 基本数据类型的静态变量

前面提到方法区用来存储一些共享数据，因此基本数据类型的静态变量名以及值存储于方法区的运行时常量池中，静态变量随类加载而加载，随类消失而消失

D. 引用数据类型的存储

堆是用来存储对象本身和数组，而引用（句柄）存放的是实际内容的地址值，因此通过上面的程序运行图，也可以看出，当我们定义一个对象时

```java
Person per=new Person();

实际上，它也是有两个过程：

Person per;//定义变量
per=new Person();//赋值

```

在执行Person per;时，JVM先在虚拟机栈中的变量表中开辟一块内存存放per变量，在执行per=new Person()时，JVM会创建一个Person类的实例对象并在堆中开辟一块内存存储这个实例，同时把实例的地址值赋值给per变量。因此可见：

第一点：对于引用数据类型的对象/数组，变量名存在栈中，变量值存储的是对象的地址，并不是对象的实际内容。看上面的【x0123】

下面就要引出 值传递和引用传递了


值传递：
在方法被调用时，实参通过形参把它的内容副本传入方法内部，此时形参接收到的内容是实参值的一个拷贝，因此在方法内对形参的任何操作，都仅仅是对这个副本的操作，不影响原始值的内容。

```java

public static void valueCrossTest(int age,float weight){
    System.out.println("传入的age："+age);
    System.out.println("传入的weight："+weight);
    age=33;
    weight=89.5f;
    System.out.println("方法内重新赋值后的age："+age);
    System.out.println("方法内重新赋值后的weight："+weight);
    }

//测试
public static void main(String[] args) {
        int a=25;
        float w=77.5f;
        valueCrossTest(a,w);
        System.out.println("方法执行后的age："+a);
        System.out.println("方法执行后的weight："+w);
}

传入的age：25
传入的weight：77.5

方法内重新赋值后的age：33
方法内重新赋值后的weight：89.5

方法执行后的age：25
方法执行后的weight：77.5

a和w作为实参传入valueCrossTest之后，无论在方法内做了什么操作，最终a和w都没变化。

```

main()压入栈中，a和w在main方法所在的栈帧中，age和weight在valueCrossTest方法所在的栈帧中，而他们的值是从a和w的值copy了一份副本而得

age和weight的改动，只是改变了当前栈帧（valueCrossTest方法所在栈帧）里的内容，当方法执行结束之后，这些局部变量都会被销毁

第一点：值传递传递的是真实内容的一个副本，对副本的操作不影响原内容，也就是形参怎么变化，不会影响实参对应的内容。

引用传递：
”引用”也就是指向真实内容的地址值，在方法调用时，实参的地址通过方法调用被传递给相应的形参，在方法体内，形参和实参指向同一块内存地址，对形参的操作会影响的真实内容。

```java

public class Person {
        private String name;
        private int age;

        public String getName() {
            return name;
        }
        public void setName(String name) {
            this.name = name;
        }
        public int getAge() {
            return age;
        }
        public void setAge(int age) {
            this.age = age;
        }
}


public static void PersonCrossTest(Person person){
        System.out.println("传入的person的name："+person.getName());
        person.setName("我是张小龙");
        System.out.println("方法内重新赋值后的name："+person.getName());
    }
//测试
public static void main(String[] args) {
        Person p=new Person();
        p.setName("我是马化腾");
        p.setAge(45);
        PersonCrossTest(p);
        System.out.println("方法执行后的name："+p.getName());
}

传入的person的name：我是马化腾
方法内重新赋值后的name：我是张小龙
方法执行后的name：我是张小龙

稍作修改

public static void PersonCrossTest(Person person){
        System.out.println("传入的person的name："+person.getName());
        person=new Person();//加多此行代码
        person.setName("我是张小龙");
        System.out.println("方法内重新赋值后的name："+person.getName());
    }

传入的person的name：我是马化腾
方法内重新赋值后的name：我是张小龙
方法执行后的name：我是马化腾

```

对象和数组是存储在Java堆区的，而且堆区是共享的，因此程序执行到main()方法中的下列代码时

main栈帧， p -> 【x02222】 | 堆，age = 45; name="马化腾"

当执行到PersonCrossTest()方法时，因为方法内有这么一行代码：person=new Person();

JVM需要在堆内另外开辟一块内存来存储new Person()，假如地址为“xo3333”，那此时形参person指向了这个地址，假如真的是引用传递，那么由上面讲到：引用传递中形参实参指向同一个对象，形参的操作会改变实参对象的改变。可以推出：实参也应该指向了新创建的person对象的地址，所以在执行PersonCrossTest()结束之后，最终输出的应该是后面创建的对象内容。

然而实际上，最终的输出结果却跟我们推测的不一样，最终输出的仍然是一开始创建的对象的内容。

第一点：由此可见：引用传递，在Java中并不存在。

但是有人会疑问：为什么第一个例子中，在方法内修改了形参的内容，会导致原始对象的内容发生改变呢？

第二点：这是因为：无论是基本类型和是引用类型，在实参传入形参时，都是值传递，也就是说传递的都是一个副本，而不是内容本身。

上面在不同的栈中的 因为值copy后的地址是一样的，所以修改有用，但是第二个例子中，PersonCrossTest栈帧中的person形参指向了新的地址（堆中），修改也就不影响原来了

综上：在Java中所有的参数传递，不管基本类型还是引用类型，都是值传递，或者说是副本传递。只是在传递过程中：

a、如果是对基本数据类型的数据进行操作，由于原始内容和副本都是存储实际值，并且是在不同的栈区，因此形参的操作，不影响原始内容。

b、如果是对引用类型的数据进行操作，分两种情况，一种是形参和实参保持指向同一个对象地址，则形参的操作，会影响实参指向的对象的内容。一种是形参被改动指向新的对象地址（如重新赋值引用），则形参的操作，不会影响实参指向的对象的内容。

注意，在C++中，引用传递这个概念是[当定义一个引用时，其实是为目标变量起一个别名，引用并不分配独立的内存空间，它与目标变量共用其内存空间]，这句很重要的是不分配内存空间，只是一个别名。所以修改形参会影响实参，它们本身就是一个东西。而java的值传递进去，是给形参开辟了一块内存存储了copy过来的地址值的，所以只要这个地址值不变，那么可以做到修改的作用，但是一旦方法中修改了地址值，无效
