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

不可变数据类型： 当该数据类型的对应变量的值发生了改变，那么它对应的内存地址也会发生改变，对于这种数据类型，就称不可变数据类型。其中基本数据类型都是不可变数据类型，例如int，如果一个int类型的数据发生改变，那么它指向了内存中的另一个地址，但是需要注意的是java缓存了所有-127-128的值。
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
