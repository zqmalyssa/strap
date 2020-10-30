---
layout: post
title: JAVA的泛型
tags: [code, java]
author-id: zqmalyssa
---

Java的泛型在JDK1.5的时候引入，泛型实现了参数化类型的概念，使代码可以用于多种类型，需要理解泛型的边界


#### 基本概述

泛型的出现，比较多的是因为需要创造容器类，容器就是一个要存放使用对象的地方。通常而言，我们只会使用容器来存储一种类型的对象，泛型的主要目的之一就是用来指定容器要持有什么类型的对象，而且由编译器来保证类型的正确性。

因此，与其使用Object，我们更喜欢暂时不指定类型，而是稍后决定具体使用什么类型，要达到这个目的，需要使用类型参数，用尖括号括住，放在类名后面，然后在使用这个类的时候，再用实际的类型替换此类型参数，看个简单的例子

```java
package com.qiming.test.genericity;

public class ClassGen<T> {

  private T a;

  public ClassGen(T a) {
    this.a = a;
  }

  public T getA() {
    return a;
  }

  public void setA(T a) {
    this.a = a;
  }

  public static void main(String[] args) {
    ClassGen<User> test = new ClassGen<User>(new User());
    User user = test.getA();
    //下面这两处编译器就报错了
//    test.setA("设置成字符串");
//    test.setA(2);
  }
}

```
现在，创建对象时必须指明持有什么类型的对象，将其置于尖括号内，你就只能在ClassGen中存入该类型（但这类子类是支持的，因为多态和泛型并不冲突）对象

#### 泛型接口

```java
package com.qiming.test.genericity.interfaces;

public interface Generator<T> {

  T next();

}
```

```java
package com.qiming.test.genericity.interfaces;

public class Fibonacci implements Generator<Integer> {

  private int count = 0;
  public Integer next() {
    return fib(count++);
  }

  private int fib(int n) {
    if (n < 2) {
      return 1;
    }
    return fib(n-2) + fib(n-1);
  }

  public static void main(String[] args) {
    Fibonacci gen = new Fibonacci();
    for (int i = 0; i < 18; i++) {
      System.out.println(gen.next() + " ");
    }
  }
}

```

#### 泛型方法

泛型也可以作用在方法上，是否拥有泛型方法与其所在的类是否是泛型没有关系，泛型方法使得该方法能够独立于类而产生变化，如果可以使用泛型方法可以取代将整个类泛型化，那就应该用泛型方法，因为可以使事情更清楚明白，static的方法而言，无法访问泛型类，所以只能用泛型方法

```java
package com.qiming.test.genericity.method;

/**
 * 泛型方法的使用例子
 */
public class GenericMethods {

  public <T> void f(T x) {
    System.out.println(x.getClass().getName());
  }

  public static void main(String[] args) {
    GenericMethods gm = new GenericMethods();
    gm.f("");
    gm.f(1);
    gm.f(1.0);
    gm.f(1.0f);
    gm.f('c');
    gm.f(gm);
  }

}

```

在使用泛型类时，必须在创建对象的时候指定类型参数的值，而使用泛型方法的时候，通常不需要指明参数类型，因为编译器会为我们找到具体的类型，这称为类型参数推断，因此我们可以像调用普通方法一样调用f()，举个可变参数的栗子

```java
public static <T> List<T> makeList(T... args) {
    List<T> result = new ArrayList<T>();
    for (T item : args) {
      result.add(item);
    }
    return result;
  }
```

```java
public static void main(String[] args) {
    List<String> ls = makeList("A");
    System.out.println(ls);
    ls = makeList("A", "B", "C");
    System.out.println(ls);
    ls = makeList("ABCDEFGHIJKLMNOPQRSTUVWXYZ".split(""));
    System.out.println(ls);
}
```

#### 泛型擦除

看个栗子

```java
package com.qiming.test.genericity;

import java.util.ArrayList;

/**
 * 泛型擦除
 */
public class Erased {

  public static void main(String[] args) {
    //很容易以为着是两个不同的class
    Class c1 = new ArrayList<String>().getClass();
    Class c2 = new ArrayList<Integer>().getClass();
    System.out.println(c1 == c2);
  }

}
```

在泛型代码内部，无法获得任何有关泛型参数类型的信息，可以知道诸如类型参数标识符和泛型类型边界这类信息，你却无法知道用来创建某个特定实例的实际的类型参数！！看下面这个例子

```java
package com.qiming.test.genericity;

public class HasF {

  public static void main(String[] args) {
    HasF hf = new HasF();
    Manipulator<HasF> manipulator = new Manipulator<HasF>(hf);
//    manipulator.manipulator();

  }

  public void f() {
    System.out.println("Hasf.f()");
  }

}

/**
 * Java编译器无法将manipulator()必须能够在obj上调用f()这一需求映射到HasF拥有f()这一事实上
 * 为了调用f()，我们必须协助泛型类，给定泛型类的边界，以此告知编译器只能接受遵循这个边界的类型！！这里就要用到extends
 * @param <T>
 */
class Manipulator<T> {
  private T obj;

  public Manipulator(T obj) {
    this.obj = obj;
  }
//
//  public void manipulator() {
//    obj.f();
//  }
}

/**
 * 这是修改后的，编译器就不报错了
 * @param <T>
 */
class Manipulator2<T extends HasF> {
  private T obj;

  public Manipulator2(T obj) {
    this.obj = obj;
  }

  public void manipulator() {
    obj.f();
  }
}
```
边界<T extends HasF>声明T必须具有类型HasF或者从HasF导出的类型，如果情况确实如此，那么就可以安全地在obj上调用f()，我们说泛型类型参数将擦除到它的第一个边界(因为可能会有多个边界)，编译器实际会把类型参数替换为它的擦除，就像上面的示例一样，T被擦除到HasF，就好像在类的声明中用HasF替换了T一样。泛型类型只有在静态类型检查期间才出现，在此之后，程序中的所有泛型类型都将被擦除，替换为它们的非泛型上界，例如，诸如List<T>这样的类型注解将被擦除为List，而普通的类型变量在未指定边界的情况下将被擦除为Object，来看些例子

```java
package com.qiming.test.genericity;

import java.util.ArrayList;

public class GenericBase<T> {

  private T element;
  public void set(T arg) {
    element = arg;
  }

  public T get() {
    return element;
  }

  public static void main(String[] args) throws Exception{
    Derived2 d2 =  new Derived2();
    Object obj = d2.get();
    d2.set(obj);

    ArrayList<Integer> list = new ArrayList<Integer>();

    list.add(1);  //这样调用 add 方法只能存储整形，因为泛型类型的实例为 Integer

//    list.add("test");

    //通过反射是可以添加其他类型元素的
    //原始类型是Object，所以通过反射我们就可以存储字符串了
    list.getClass().getMethod("add", Object.class).invoke(list, "asd");

    for (int i = 0; i < list.size(); i++) {
      System.out.println(list.get(i));
    }

    /**不指定泛型的时候*/
    int i = GenericBase.add(1, 2); //这两个参数都是Integer，所以T为Integer类型
    Number f = GenericBase.add(1, 1.2); //这两个参数一个是Integer，以风格是Float，所以取同一父类的最小级，为Number
    Object o = GenericBase.add(1, "asd"); //这两个参数一个是Integer，以风格是Float，所以取同一父类的最小级，为Object

    /**指定泛型的时候*/
    int a = GenericBase.<Integer>add(1, 2); //指定了Integer，所以只能为Integer类型或者其子类
//    int b = GenericBase.<Integer>add(1, 2.2); //编译错误，指定了Integer，不能为Float
    Number c = GenericBase.<Number>add(1, 2.2); //指定为Number，所以可以为Integer和Float

  }

  //这是一个简单的泛型方法
  public static <T> T add(T x,T y){
    return y;
  }
}

```
因为种种原因，Java不能实现真正的泛型，只能使用类型擦除来实现伪泛型，这样虽然不会有类型膨胀问题，但是也引起来许多新问题，所以，SUN对这些问题做出了种种限制，避免我们发生各种错误。

1、先检查，再编译以及编译的对象和引用传递问题

既然说类型变量会在编译的时候擦除掉，那为什么我们往 ArrayList 创建的对象中添加整数会报错呢？不是说泛型变量String会在编译的时候变为Object类型吗？为什么不能存别的类型呢？既然类型擦除了，如何保证我们只能使用泛型变量限定的类型呢？
  - 这是因为Java编译器是通过先检查代码中泛型的类型，然后在进行类型擦除，再进行编译。

```java
public void test1() {
    ArrayList<String> list1 = new ArrayList(); //第一种情况
    list1.add("1"); //编译通过
    list1.add(1); //编译错误
    String str1 = list1.get(0); //返回类型就是String

    ArrayList list2 = new ArrayList<String>(); //第二种情况
    list2.add("1"); //编译通过
    list2.add(1); //编译通过  
    Object object = list2.get(0); //返回类型就是Object

    new ArrayList<String>().add("11"); //编译通过
    new ArrayList<String>().add(22); //编译错误

    String str2 = new ArrayList<String>().get(0); //返回类型就是String
  }
```
看上面这个例子，第一种情况跟第二种情况都行，不过在第一种情况，可以实现与完全使用泛型参数一样的效果，第二种则没有效果。因为类型检查就是编译时完成的，new ArrayList()只是在内存中开辟了一个存储空间，可以存储任何类型对象，而真正设计类型检查的是它的引用，因为我们是使用它引用list1来调用它的方法，比如说调用add方法，所以list1引用能完成泛型类型的检查。而引用list2没有使用泛型，所以不行。类型检查就是针对引用的，谁是一个引用，用这个引用调用泛型方法，就会对这个引用调用的方法进行类型检测，而无关它真正引用的对象。在Java中，像下面形式的引用传递是不允许的:

```java
ArrayList<String> list1 = new ArrayList<Object>(); //编译错误  
ArrayList<Object> list2 = new ArrayList<String>(); //编译错误
```
先看第一种情况，将第一种情况拓展成下面的形式：

```java
ArrayList<Object> list1 = new ArrayList<Object>();  
list1.add(new Object());  
list1.add(new Object());  
ArrayList<String> list2 = list1; //编译错误
```
实际上，在第4行代码的时候，就会有编译错误。那么，我们先假设它编译没错。那么当我们使用list2引用用get()方法取值的时候，返回的都是String类型的对象（上面提到了，类型检测是根据引用来决定的），可是它里面实际上已经被我们存放了Object类型的对象，这样就会有ClassCastException了。所以为了避免这种极易出现的错误，Java不允许进行这样的引用传递。（这也是泛型出现的原因，就是为了解决类型转换的问题，我们不能违背它的初衷）。

再看第二种情况，将第二种情况拓展成下面的形式：

```java
ArrayList<String> list1 = new ArrayList<String>();  
list1.add(new String());  
list1.add(new String());

ArrayList<Object> list2 = list1; //编译错误
```
没错，这样的情况比第一种情况好的多，最起码，在我们用list2取值的时候不会出现ClassCastException，因为是从String转换为Object。可是，这样做有什么意义呢，泛型出现的原因，就是为了解决类型转换的问题。我们使用了泛型，到头来，还是要自己强转，违背了泛型设计的初衷。所以java不允许这么干。再说，你如果又用list2往里面add()新的对象，那么到时候取得时候，我怎么知道我取出来的到底是String类型的，还是Object类型的呢？

所以，要格外注意，泛型中的引用传递的问题。

2、自动类型转换

因为类型擦除的问题，所以所有的泛型类型变量最后都会被替换为原始类型。既然都被替换为原始类型，那么为什么我们在获取的时候，不需要进行强制类型转换呢？
  - 看下面的Get，可以看到，在return之前，会根据泛型变量进行强转。假设泛型类型变量为Date，虽然泛型信息会被擦除掉，但是会将(E) elementData[index]，编译为(Date)elementData[index]。所以我们不用自己进行强转。当存取一个泛型域时也会自动插入强制类型转换。

```java
public E get(int index) {  

    RangeCheck(index);  

    return (E) elementData[index];  

}
```

3、类型擦除与多态的冲突和解决方法

有个泛型类

```java
package com.qiming.test.genericity;

/**
 * 类型擦除与多态的冲突和解决方法
 * @param <T>
 */
public class Pair<T> {

  private T value;

  public T getValue() {
    return value;
  }

  public void setValue(T value) {
    this.value = value;
  }
}
```

然后我们用一个子类去继承它

```java
package com.qiming.test.genericity;

import java.util.Date;

public class DateInter extends Pair<Date> {

  @Override
  public Date getValue() {
    return super.getValue();
  }

  @Override
  public void setValue(Date value) {
    super.setValue(value);
  }
}

```
在这个子类中，我们设定父类的泛型类型为Pair<Date>，在子类中，我们覆盖了父类的两个方法，我们的原意是这样的：将父类的泛型类型限定为Date，那么父类里面的两个方法的参数都为Date类型。从override看，这是个重写，没有问题，但是想想，父类擦除后是Object，子类是Date，这不应该是个重载吗

以下这个案例也是神奇，编译器时不报错的。。什么原因？
```java
public void testReload(Object obj) {
    System.out.println(obj);
  }

  public void testRelaod(Object data) {
    System.out.println(data);
  }

  public void testRelaod(Date date) {
    System.out.println(date);
  }
```
先回到上面的`DateInter`，用`javap -c className`的方式反编译下`DateInter`子类的字节码，结果如下：

```java
class com.tao.test.DateInter extends com.tao.test.Pair<java.util.Date> {  
  com.tao.test.DateInter();  
    Code:  
       0: aload_0  
       1: invokespecial #8                  // Method com/tao/test/Pair."<init>":()V  
       4: return  

  public void setValue(java.util.Date);  //我们重写的setValue方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: invokespecial #16                 // Method com/tao/test/Pair.setValue:(Ljava/lang/Object;)V  
       5: return  

  public java.util.Date getValue();    //我们重写的getValue方法  
    Code:  
       0: aload_0  
       1: invokespecial #23                 // Method com/tao/test/Pair.getValue:()Ljava/lang/Object;  
       4: checkcast     #26                 // class java/util/Date  
       7: areturn  

  public java.lang.Object getValue();     //编译时由编译器生成的巧方法  
    Code:  
       0: aload_0  
       1: invokevirtual #28                 // Method getValue:()Ljava/util/Date 去调用我们重写的getValue方法;  
       4: areturn  

  public void setValue(java.lang.Object);   //编译时由编译器生成的巧方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: checkcast     #26                 // class java/util/Date  
       5: invokevirtual #30                 // Method setValue:(Ljava/util/Date; 去调用我们重写的setValue方法)V  
       8: return  
}
```
从编译的结果来看，我们本意重写setValue和getValue方法的子类，竟然有4个方法，其实不用惊奇，最后的两个方法，就是编译器自己生成的桥方法。可以看到桥方法的参数类型都是Object，也就是说，子类中真正覆盖父类两个方法的就是这两个我们看不到的桥方法。而打在我们自己定义的setvalue和getValue方法上面的@Oveerride只不过是假象。而桥方法的内部实现，就只是去调用我们自己重写的那两个方法。

所以，虚拟机巧妙的使用了桥方法，来解决了类型擦除和多态的冲突。

不过，要提到一点，这里面的setValue和getValue这两个桥方法的意义又有不同。

setValue方法是为了解决类型擦除与多态之间的冲突。

而getValue却有普遍的意义，怎么说呢，如果这是一个普通的继承关系：

那么父类的getValue方法如下：

```java
public ObjectgetValue() {  
    return super.getValue();  
}
```
而子类重写的方法是：
```java
public Date getValue() {  
    return super.getValue();  
}
```
其实这在普通的类继承中也是普遍存在的重写，这就是协变。关于协变：。。。。。。

4、泛型变量不能是基础数据类型

不能用类型参数替换基本类型。就比如，没有ArrayList<double>，只有ArrayList<Double>。因为当类型擦除后，ArrayList的原始类型变为Object，但是Object类型不能存储double值，只能引用Double的值。

5、运行时类型查询

比如

```java
ArrayList<String> arrayList = new ArrayList<String>();
```
因为类型擦除之后，ArrayList<String>只剩下原始类型，泛型信息String不存在了。

那么，运行时进行类型查询的时候使用下面的方法是错误的

```java
if( arrayList instanceof ArrayList<String>)
```

6、泛型在静态方法和静态类中的问题

泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数

```java
public class Test2<T> {    
    public static T one;   //编译错误    
    public static  T show(T one){ //编译错误    
        return null;    
    }    
}
```
因为泛型类中的泛型参数的实例化是在定义对象的时候指定的，而静态变量和静态方法不需要使用对象来调用。对象都没有创建，如何确定这个泛型参数是何种类型，所以当然是错误的。但是要注意区分下面的一种情况：

```java
public class Test2<T> {    

    public static <T> T show(T one){ //这是正确的    
        return null;    
    }    
}
```
因为这是一个泛型方法，如上文所述，在泛型方法中使用的T是自己在方法中定义的 T，而不是泛型类中的T

#### 泛型边界
因为擦除机制移除了类型信息，所以若是没有给类型参数指定边界，那么调用的方法就只能是Object的方法。若是将类型参数限制为某个类的子集，那么我们就用这些子集来调用这个类的方法。为了执行这种限制，Java泛型重用了extends关键字（需要注意与继承关系中的含义区分）

```java
package com.qiming.test.genericity;

import java.awt.Color;

interface HasColor {
  java.awt.Color getColor();
}

class Colored <T extends HasColor>{
  T item;
  public Colored( T item) { this.item = item; }
  T getItem() {return item;}
  //有了边界 允许调用getColor()方法
  Color color(){ return item.getColor();}
}

class Dimension { public int x, y, z;}

//多边界 类要放在第一个 接口放在后面
//class ColoredDimension<T extends HasColor & Dimension>
class ColoredDimension <T extends Dimension & HasColor>{
  //...
}

interface Wight{int wight();}

//拥有多个边界的泛型类  多边界只能有一个具体类  但是可以有多个接口
class Solid <T extends Dimension & HasColor & Wight>{
  T item;
  public Solid(T item) { this.item = item; }
  T getItem() { return item;}
  Color color(){ return item.getColor();}
  int getX() {return item.x; }
  int getY() {return item.y; }
  int getZ() {return item.z; }
  int weight() {return item.wight(); }
}

class Bounded extends Dimension implements HasColor, Wight{
  public int wight() { return 0; }
  public Color getColor() { return null;}
}

public class BasicBounds {
  public static void main(String[] args) {
    Solid<Bounded> solid = new Solid<>(new Bounded());
    solid.color();
    solid.getX();
    solid.weight();
  }
}

```
泛型类的类型参数被限制为多边界时，具体类要放在第一个，接口放在后面
多边界时，具体类只能有一个，可以有多个接口


#### 泛型通配符

看一个数组的例子

```java
package com.qiming.test.genericity;

class Fruit{}
class Apple extends Fruit{}
class Jonathan extends Apple{}
class Orange extends Fruit{}

public class CovriantArrays {
  public static void main(String[] args) {
    Fruit[] fruit = new Apple[10];
    fruit[0] = new Apple();
    fruit[1] = new Jonathan();

    /**
     * 这边拿出去就会报错
     */
    try {
      fruit[2] = new Fruit();
    }catch (Exception e) {
      System.out.println(e);
    }

    /**
     * 这边拿出去就会报错
     */
    try {
      fruit[3] = new Orange();
    } catch (Exception e) {
      System.out.println(e);
    }

    System.out.println(fruit.getClass().getSimpleName());
  }
}
```
我们将Apple数组赋值给Fruit数组，是因为Apple也是一种Fruit。我们将Fruit放到Fruit数组中，这是被编译器允许的，因为引用类型就是Fruit。向Fruit中添加Orange也是被允许的，因为Orange也是一种Fruit。虽然在编译时期，这种赋值是被允许的，但是在运行时期却抛出了异常。原因是因为:

运行时期数组机制知道它处理的是Apple[]，添加除Apple以及Apple子类之外的对象都是不允许的。数组对象可以保留它们包含的对象类型的规则，对数组的这种赋值，将在运行时期才可以看出错误。但是泛型的主要目标之一就是将这种错误检查移入到编译期，当我们使用泛型容器代替以上数组时：

```java
List<Fruit> fList = new ArrayList<Apple>();
```
编译时的报错信息为：不能将一个Apple容器赋值给一个Fruit容器。但是更准确的说法是：不能将一个涉及Apple的泛型赋值给一个涉及Fruit的泛型。我们讨论的是容器的类型，不是容器持有的类型，所以Apple的List不是Fruit的List。与数组不同，泛型没有內建的协变类型。数组中Apple可以赋值给Fruit，是因为编译器知道Apple是Fruit的协变类型，因此可以向上转型。泛型中，若是想在两个类之间建立类似这种向上转型的关系，就需要使用通配符（即类型参数中的?）

```java
List<? extends Fruit> fList = new ArrayList<Apple>();
fList.add(new Apple());
fList.add(new Fruit());
fList.add(new Object());

Fruit f = fList.get(0);
```

查看List的实现源码，我们可以发现add()的参数会变成? extends Fruit，因此编译器不能知道需要Fruit的哪个子类型，因此它不会接受任何的Fruit。编译器将直接拒绝对参数列表中涉及通配符的方法的调用（例如add())

**超类型通配符**

若是我们想向基类型列表中写入子类型，完成上述add()方法的功能，那么我们可以使用超类型通配符。这里，可以声明通配符是由某个特定类的任何基类来界定的，方法是指定<? super MyClass> 或者使用类型参数<? super T>（但是不能对泛型类型参数给出一个超类型边界，即不能声明<T super MyClass>）。这样，我们便可以安全地传递一个类型对象到泛型类型中。因此，有了超类型通配符，我们可以做如下插入

```java
static void writeTo(List<? super Apple> apples){
    apples.add(new Apple());
    apples.add(new Jonathan());
    // apples.add(new Fruit()); //Error
}
```
我们可以向apples中添加Apple或者Apple的子类型是安全的。超类型边界放松了在可以向方法传递参数上所做的限制！

```java
package com.qiming.test.genericity;

import java.util.ArrayList;
import java.util.List;

public class GenericWriting {

  static List<Apple> apples = new ArrayList<Apple>();
  static List<Fruit> fruits = new ArrayList<Fruit>();

  static <T> void writeExact(List<T> list, T item) {
    System.out.println(item.getClass().getSimpleName());
    list.add(item);
  }

  //在“精确”类型下 也可以向fruit中添加对象
  static void f1() {
    writeExact(fruits, new Fruit());
    writeExact(fruits, new Apple());
    writeExact(fruits, new Orange());
//      writeExact(fruits, new Object()); //Error
  }

  static <T> void writeWithWildcard(List<? super T> list, T item) {
    System.out.println(item.getClass().getSimpleName());
    list.add(item);
  }

  static void f2() {
    writeWithWildcard(fruits, new Fruit());
    writeWithWildcard(fruits, new Apple());
    writeWithWildcard(fruits, new Orange());
//      writeWithWildcard(fruits, new Object()); //Error
  }

  public static void main(String[] args) {
    f1();
    System.out.println("--------------");
    f2();
  }

}

```

```java
package com.qiming.test.genericity;

import java.util.Arrays;
import java.util.List;

public class GenericReading {

  //Arrays.asList()生成大小不可变的列表
  static List<Apple> apples = Arrays.asList(new Apple());
  static List<Fruit> fruits = Arrays.asList(new Fruit());

  //使用“精确”的泛型
  static class Reader<T> {
    T readExact(List<T> list) {
      System.out.println(list.get(0).getClass().getSimpleName());
      return list.get(0);
    }
  }

  static void f1() {
    Reader<Fruit> fruitReader = new Reader<Fruit>();
    Fruit f = fruitReader.readExact(fruits);
//      Fruit a = fruitReader.readExact(apples);   //Error
  }

  //协变
  static class CovariantReader<T> {
    //可以接受T类型或者是T导出的类型
    T readCovariant(List<? extends T> list) {
      System.out.println(list.get(0).getClass().getSimpleName());
      return list.get(0);
    }
  }

  static void f2() {
    CovariantReader<Fruit> fReader = new CovariantReader<>();
    Fruit f = fReader.readCovariant(fruits);
    Fruit a = fReader.readCovariant(apples);
  }

  public static void main(String[] args) {
    f1();
    System.out.println("---");
    f2();
  }

}

```
上面这个例子中有协变的例子

#### 泛型的自限定

```java
package com.qiming.test.genericity;

class SelfBounded<T extends SelfBounded<T>>{
  T element;
  public SelfBounded<T> set(T arg) {
    element = arg;
    return this;
  }

  public T get() {
    return element;
  }
}

class A extends SelfBounded<A>{}
class B extends SelfBounded<A>{}

class C extends SelfBounded<C>{
  C setAndGet(C arg) {
    set(arg);
    return get();
  }
}
//The type D is not a valid substitute for the bounded parameter <T extends SelfBounded<T>> of the type SelfBounded<T>
class D{}
//class E extends SelfBounded<D>{}

class F extends SelfBounded{}

public class SelfBounding {
  public static void main(String[] args) {
    A a = new A();
    a.set(new A());
    a = a.set(new A()).get();
    a = a.get();
    C c = new C();
    c = c.setAndGet(new C());
  }
}
```
自限定要求的就是在继承关系中，像下面这样使用这个类：

```java
class A　extends SelfBouned<A>
```
那么我们又想知道自限定的参数有什么作用呢？它可以保证参数类型必须与正在被定义的类相同！我们从代码中可以看出虽然可以使用，B虽然可以继承从SelfBounded导出的A，但是B类中的类型参数都是为A类。A类的那种继承为最常用的用法。对E类进行定义说明不能使用不是SelfBounded的类型参数。F可以编译，不会有任何警告，说明自限定惯用法不是可强制执行的。

自限定类型只能强制作用于继承关系，如果使用了自限定，就应该了解这个类的所有类型参数将与使用这个参数的类具有相同类型。即类型参数与类具有相同类型。


#### 参数协变
自限定类型的价值在于：可以产生协变参数类型（方法参数类型会随着子类而变化）。

```java
package com.qiming.test.genericity;

/**
 * 使用自限定类型，方法参数类型会随着子类而变化
 * @param <T>
 */
class GenericGetter<T extends GenericGetter<T>>{
  T element;
  void set(T element) { this.element = element; }
  T get() { return element; }
}

class Getter extends GenericGetter<Getter>{
}

public class GenericAndReturnTypes {
  static void test(Getter g) {
    Getter result = g.get();
    GenericGetter genericGetter = g.get();
  }

  public static void main(String[] args) {
    Getter getter = new Getter();
    test(getter);
  }
}

```

但是在非泛型代码中，参数类型却不可以随子类变化而变化。

```java
package com.qiming.test.genericity;

/**
 * 在非泛型的代码中，参数类型却不可以随子类变化而变化
 */
class Base{}
class Derived extends Base{}

class OrdinarySetter{
  void set(Base base) {
    System.out.println("OrdinarySetter.set(Base)");
  }
}

class DerivedSetter extends OrdinarySetter{
  void set(Derived derived) {
    System.out.println("DerivedSetter.set(Derived)");
  }
}

public class OrdinaryArguments {
  public static void main(String[] args) {
    Base base = new Base();
    Derived derived = new Derived();
    DerivedSetter ds = new DerivedSetter();

    ds.set(base);
    ds.set(derived);
  }
}

```
ds.set(base);和ds.set(derived);都可以可以的，是因为DerivedSetter.set()没有重写OrdinarySetter.set()中的方法，而是重载了。于是DerivedSetter中含有两个set方法。ds.set(base);调用的是父类OrdinarySetter的set的。

但是使用自限定类型，在导出类中就只会有一个方法，并且这个方法接受导出类型而不是基类型为参数！！！举个栗子

```java
interface SelfBoundSetter<T extends SelfBoundSetter<T>>{
    void set(T arg);
}

interface Setter extends SelfBoundSetter<Setter>{}

public class SelfBoundingAndCovariantAruguments {
    void test(Setter s1, Setter s2, SelfBoundSetter sbs) {
        s1.set(s2);
        //The method set(Setter) in the type SelfBoundSetter<Setter>
        //is not applicable for the arguments (SelfBoundSetter)
        //s1.set(sbs);
    }
}
```
以上是使用自限定类，基类型就不可以传入到子类型方法中
若是不使用自限定类型，而使用普通泛型，则子类中就是重载基类的方法，结果就像在OrdinaryArguments.java中一样。可以看出不使用自限定类型将重载参数，使用自限定类型将只能获得方法的一个版本，它将接受确切的参数类型。
