---
layout: post
title: Java设计模式
tags: [code, java, design]
author-id: zqmalyssa
---

Java设计模式是Java学习中的重中之重，也是掌握的难点，需要持续更进

参考《Head First设计模式》，《Effective Java》，《数据结构与算法JAVA》


#### **策略模式**

**策略模式**定义了算法族，分别封装起来，让它们之间可以相互替换，此模式让算法的变化独立于使用算法的客户

1. 找出应用中可能需要变化之处，把它们独立出来，不要和那些不需要变化的代码混在一起
2. 针对接口编程，而不是针对实现编程，关键在于多态
3. 某个类将自己本身的一些行为委托(delegate)给别人处理
4. 当将两个类结合起来使用，就是组合(composition)，多用组合，少用继承

![strategy]({{ "/assets/img/designpattern/strategy.png" | relative_url}})

**案例**

如果有一个动作冒险游戏，你将看到代表游戏角色的类和角色可以使用的武器行为的类，每个角色一次只能使用一种武器，但是可以在游戏的过程中换武器

书中为鸭子案例

《EJ》中指出Comparator接口实现的是一个策略接口，这个要关注一下

《DSAA》中写数组的数据结构时定义了一个strategy，这个变量名字一看就想到了策略模式，由其他的实现类去实现Strategy接口，与正式的数组代码解耦

#### **观察者模式**

**观察者模式**出版者、主题(Subject)，订阅者、观察者(Observer)，两种结合起来就是观察者模式，观察者模式定义了对象之间的一对多依赖，这样一来，当一个对象改变状态时，它的所有依赖者都会收到通知并自动更新

1. 观察者提供一种对象设计，让主题和观察者之间松耦合
2. 可以独立的复用主题或者观察者，有新的观察者出现，主题的代码不需要修改，改变主题或者观察者任一方，并不会影响到另一方
3. 主题这相当于在发生改变的时候，主动的去通知了所有观察者这些改变，这是"推"，那么反过来，其实也可以观察者主动去"拉"，拉就是主题中提供get方法

![observer]({{ "/assets/img/designpattern/observer.png" | relative_url}})

**案例**

Java的util包中有Observable的自带观察者模式，Swing API中的JButton也是应用观察者模式去响应事件的，观察者的代表模式是MVC

书中为气象局数据案例

#### **装饰者模式**

**装饰者模式**动态的将责任附加到对象上，若要扩展功能，装饰者提供了比继承更有弹性的替代方案

1. 类应该对扩展开发，对修改关闭
2. 装饰者和被装饰对象有相同的超类型，可以用一个或多个装饰者包装一个对象
3. 既然装饰者和被装饰对象有相同的超类型，在任何需要原始对象(被包装的)的场合，可以用装饰过的对象代替它
4. 装饰者可以在所委托被装饰者的行为之前与/或之后，加上自己的行为，以达到特定的目的
5. 对象可以在任何时候被装饰，所以可以运行时动态的，不限量的用你喜欢的装饰者来装饰对象

![decorator]({{ "/assets/img/designpattern/decorator.png" | relative_url}})

**案例**

Java的IO包内使用了很多装饰者模式，FilterInputStream是装饰的"组件"，BufferedInputStream是一个具体的"装饰者"，LineNumberInputStream也是一个具体的装饰者

书中为星巴克咖啡案例

还有EJ中也提到可以用装饰者模式，这不是一种继承，而是一种复合，比如用来扩展Set集合，增加计数器

#### **工厂模式**

**工厂模式**工厂方法模式定义了一个创建对象的接口，但由子类决定要实例化的类是哪一个，工厂方法让类把实例化推迟到子类

**工厂模式**抽象工厂模式提供一个接口，用于创建相关或依赖对象的家族，而不需要明确指定具体类

1. 设计模式中，实现一个接口并不一定表示写一个类，并利用implement关键字实现，它泛指实现某个超类型(可以是类或接口)的某个方法
2. 工厂中原本是由一个对象负责所有具体类的实例化，转变后，变成由一群子类来负责实例化
3. 工厂方法用来处理对象的创建，并将这样的行为封装在子类中，关于超类的代码就和子类对象创建代码解耦了
4. 依赖倒置原则，要依赖抽象，不要依赖具体类，不要让高层组件依赖低层组件
	- 变量不可以持有具体的类的引用，如果new，就会持有具体类的引用，你可以改用工厂来避开这种做法
	- 不要让类派生自具体类，如果派生自具体类，你就会依赖具体类，请派生自一个抽象（接口或抽象类）
	- 不要覆盖基类中已实现的方法，如果覆盖，就说明基类不是真正适合被继承的
5. 工厂方法和抽象工厂有区别，有的时候抽象工厂实现的时候会用到工厂方法

**案例**

有一个不叫设计模式的，简单工厂模式，简单工厂是什么样呢

```java
public class PizzaStore {

	SimplePizzaFactory factory;

	public PizzaStore(SimplePizzaFactory factory) {
		this.factory = factory;
	}

	public Pizza orderPizza(String type) {
		Pizza pizza;

		//将原来在这边new Pizza的过程抽出去，放到SimplePizzaFactory里面
		pizza = factory.createPizza(type);

		pizza.prepare();
		pizza.bake();
		pizza.cut();
		pizza.box();
		return pizza;

	}

}

public class SimplePizzaFactory {

	public Pizza createPizza(String type) {
		Pizza pizza = null;

		if (type.equals("cheese")) {
			pizza = new CheesePizza();
		} else if (type.equals("pepperoni")) {
			pizza = new PepperoniPizza();
		} else if (type.equals("clam")) {
			pizza = new ClamPizza();
		} else if (type.equals("veggie")) {
			pizza = new VeggiePizza();
		}

		return pizze;
	}

}

```


书中为披萨店案例

这边要提到的是还有一个**静态工厂方法**，这不是一个设计模式，这是一种替代构造器的方式(不要先考虑公有构造器)
1. 静态工厂方法可以写自己的名字
2. 不必每次调用他们的时候都创建一个新对象
3. 可以返回原返回类型的任何子类型的对象
4. 在创建参数化类型实例的时候，它们使代码变得更简洁

```java

public static <K,V> HashMap<K,V> newInstance() {
	return new HashMap<K,V>();
}

Map<String, List<String>> m = HashMap.newInstance();

```

#### **单例模式**

**单例模式**确保一个类只有一个实例，并提供一个全局访问点

1. 单例模式在多线程下同步的问题，解决方法，1是使用synchronized关键字包起来getInstance的方法，2是使用"急切"实例化法，依赖JVM在加载这个类时马上创建此唯一的单件实例，JVM保证在任何线程访问静态变量之前，一定先创建此实例，3是双重检查加锁，首先检查是否实例已经创建，如果尚未创建，"才"进行同步，只有第一次会同步(因为1中的synch也只是在第一次才起到作用，后面即使你多线程，也无所谓了)，第三种是比较好的方法

**案例**

```java
package com.qiming.designpattern.singleton;

public class DCLSingleton {

  //volatile关键字是为了解决多线程环境下指令重排
  private volatile static DCLSingleton dclSingleton;

  private DCLSingleton() {}

  //多线程环境中，不用在方法上加syn，只有第一次执行此方法，才需要真正同步
  public static DCLSingleton getInstance() {
    //双重锁是为了解决速度问题
    if (dclSingleton == null) {
      synchronized (DCLSingleton.class) {
        if (dclSingleton == null) {
          dclSingleton = new DCLSingleton();
        }
      }
    }
    return dclSingleton;
  }

}


```

```java
package com.qiming.designpattern.singleton;

/**
 * 延迟初始化方案，Initialization On Demand Holder idiom
 *
 */
public class InstanceFactory {

  private static class InstanceHolder {
    private static Instance instance = new Instance();
  }

  public static Instance getInstance() {
    return InstanceHolder.instance; //这里将导致InstanceHolder类被初始化
  }

}

class Instance {

}

```

#### **命令模式**

**命令模式**将"请求"封装成对象，以便使用不同的请求，队列或者日志来参数化其他对象，命令模式也支持可撤销的操作

1. 请求调用者和请求接受者之间的解耦
2. 一个命令对象通过在特定接受者上绑定一组动作来封装一个请求，要达到这一点，命令对象将动作和接收者包进对象中

**案例**

可以应用在队列，日志等场景中

书中为遥控器编程案例

#### **适配器模式与外观模式**

**适配器模式**将一个类的接口，转换成客户期望的另一个接口，适配器让原本接口不兼容的类可以合作无间

**外观模式**提供了一个统一的接口，用来访问子系统中的一群接口，外观定义了一个高层接口，让子系统更容易使用

1. 有对象适配器和类适配器之分，对象是组合的方式，而类的多继承的方式
2. 外观不只是简化了接口，也将客户从组件的子系统解耦，外观和适配器都可以包装许多类，但是外观的意图是简化接口，而适配器的意图是将接口转换成不同接口
3. 最少只是原则，只和你的密友谈话

**案例**

书中为火鸡、鸭，家庭影院案例

#### **模板方法模式**

**模板方法模式**在一个方法中定义一个算法的骨架，而将一些步骤延迟到子类中。模板方法使得子类可以在不改变算法结构的情况下，重新定义算法中的某些步骤

1. 模板方法被声明为final，以免子类改变这个算法的顺序
2. 可以定义抽象方法，在模板方法中使用，但是由子类去实现
3. 也可以有"默认不做事的方法"，我们称这个方法为"hook"，钩子，子类可以视情况覆盖他们
4. 别调用(打电话给)我们，我们会调用(打电话给)你
5. 模板和策略的比较，策略是组合思维，模板呢，依赖程度比策略高

**案例**

Java数组中的排序是应用到了模板方法的思想，实现确也有点像策略模式，sort方法被实现成一个静态方法

书中为泡咖啡、泡茶案例

#### **迭代器与组合模式**

**迭代器模式**提供一种方法顺序访问一个聚合对象中的各个元素，而又不暴露其内部的表示

**组合模式**允许你将对象组合成树形结构来变现"整体/部分"层次结构。组合能让客户以一致的方式处理个别对象以及对象组合

1. ArrayList和数组都可以使用迭代器进行遍历
2. 一个类应该只有一个引起变化的原因，尽量让每个类保持单一责任
3. 组合和迭代器之间没有什么转换，两者倒是可以合作无间，可以再组合的实现中使用迭代器
4. 组合一般应用于树形结构

**案例**

Java中的ArrayList等集合类跟数组可以模拟迭代器模式

书中为菜单跟子菜单和菜单项的案例

#### **状态模式**

**状态模式**允许对象在内部状态改变时改变它的行为，对象看起来好像修改了它的类

1. 看看策略模式和状态模式的类图比较，两者是很相似的
2. 扩展状态会很方便，对修改关闭


**案例**

书中为糖果机案例

#### **代理模式**

**代理模式**为另一个对象提供一个替身或占位符以控制对这个对象的访问

1. 远程代理和本地代表活在不同的JVM中
2. 这边就需要懂点**RMI**了
3. 虚拟代理，正在加载CD封面，加载完后显示封面
4. 代理跟装饰者有点像，但本质是不同的
5. 这里还有动态代理，代理包含两个类，proxy和invocation，保护代理
6. 动态代理之所以被称为动态，是因为运行时才将它的类创建出来，代码开始执行时，还没有proxy类，它是根据需要从你传入的接口集创建的


**案例**

书中为糖果机编程远程服务的案例，动态代理为人物评分案例

**补充**

Java的动态代理分两种，JDK自带的，和CGLIB的Spring的AOP原理是基于动态代理来实现的

1. 基于JDK的动态代理

基于JDK的动态代理就需要知道两个类：1.InvocationHandler（接口）、2.Proxy（类）

第一步需要编写一个接口

```java
public interface JDKSubject {

  void hello(String val);

}
```

第二步需要实现这个接口

```java
public class JDKSubjectImpl implements JDKSubject{

  public void hello(String val) {
    System.out.println("hello" + val);
  }
}
```

第三步创建Impl的代理类

```java
public class JDKSubjectProxy implements InvocationHandler {

  private JDKSubject jdkSubject;

  public JDKSubjectProxy(JDKSubject jdkSubject) {
    this.jdkSubject = jdkSubject;
  }

  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    System.out.println("--------------begin-------------");
    Object invoke = method.invoke(jdkSubject, args);  //利用反射调用类里面的实际方法，method是那个方法，invoke是方法的返回值
    System.out.println("--------------end-------------");
    return invoke;
  }
}
```

进行测试
```java
public class JDKMain {
  public static void main(String[] args) {
    JDKSubject subject = new JDKSubjectImpl();
    InvocationHandler subjectProxy = new JDKSubjectProxy(subject);
    JDKSubject proxyInstance = (JDKSubject) Proxy
        .newProxyInstance(subjectProxy.getClass().getClassLoader(), subject.getClass().getInterfaces(), subjectProxy);
	//第一个参数是代理类的类加载器，第二个参数是被代理类的接口，如果有多个就是数组形式传入，第三个参数是代理类实例
    proxyInstance.hello("world");
  }
}
```
2. 基于CGLib的代理

因为基于JDK的动态代理一定要继承一个接口，而绝大部分情况是基于**POJO类**的动态代理，注意这里是作用在类上，那么CGLIB就是一个很好的选择，在Hibernate框架中PO的字节码生产工作就是靠CGLIB来完成的。还是先看代码，引入cglib和asm包

第一步创建代理类

```java
public class CGlibSubject {

  public void sayHello(){
    System.out.println("hello world");
  }

}
```
第二步如果直接对这个类创建对象，那么调用sayHello方法，控制台就会输出hello world，现在我们还是要对输出添加前置和后置的log输出。来打印输出前和输出后的时间。实现MethodInterceptor接口，对方法进行拦截处理

```java
public class CGlibInterceptor implements MethodInterceptor {

  public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy)
      throws Throwable {
    //d是被代理对象的实例，methodProxy是代理方法
    System.out.println("begin time -----> "+ System.currentTimeMillis());
    Object o1 = methodProxy.invokeSuper(o, objects);  //调用被拦截的方法，不要使用invoke，会出现OOM的情况
    System.out.println("end time -----> "+ System.currentTimeMillis());
    return o1;
  }
}
```
第三步创建被代理类，利用Enhancer来生产被代理类，这样可以拦截方法，对方法进行前置和后置log的添加

```java
public class CGlibMain {

  public static void main(String[] args) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(CGlibSubject.class);
    enhancer.setCallback(new CGlibInterceptor());
    CGlibSubject cGsubject = (CGlibSubject) enhancer.create();
    cGsubject.sayHello();
  }

}
```

3. 静态代理

静态代理也要知道一下，看个sample

```java
package com.qiming.test.proxy;

/**
 * 演示静态代理
 */
public interface Person {

  //交作业
  void giveTask();

}

```
```java
package com.qiming.test.proxy;

/**
 * 被代理的类
 */
public class Student implements Person{

  private String name;

  public Student(String name) {
    this.name = name;
  }

  @Override
  public void giveTask() {
    System.out.println(name + " 交语文作业");
  }
}

```

```java
package com.qiming.test.proxy;

/**
 * 代理类
 */
public class StudentsProxy implements Person{

  //被代理的学生
  Student stu;

  public StudentsProxy(Person stu) {
    //只代理学生对象
    if (stu.getClass() == Student.class) {
      this.stu = (Student)stu;
    }
  }

  //代理交作业，调用被代理学生的交作业行为，可以在里面进行增强
  @Override
  public void giveTask() {
    System.out.println("强哥最近很猛");
    stu.giveTask();
    System.out.println("老师您检查一下");
  }
}

```

测试一下

```java
package com.qiming.test.proxy;

public class TestStaticProxyMain {

  public static void main(String[] args) {

    //被代理的学生强哥，他的作业上交有代理对象monitor完成
    Person qiangge = new Student("强哥");

    //生成代理对象，并将林浅传给代理对象
    Person monitor = new StudentsProxy(qiangge);

    //班长代理交作业
    monitor.giveTask();

  }

}

```
这里并没有直接通过强哥（被代理对象）来执行交作业的行为，而是通过班长（代理对象）来代理执行了。这就是代理模式。代理模式就是在访问实际对象时引入一定程度的间接性，这里的间接性就是指不直接调用实际对象的方法，那么我们在代理过程中就可以加上一些其他用途。比如班长在帮强哥交作业的时候想告诉老师最近强哥的进步很大，就可以轻松的通过代理模式办到。

动态代理和静态代理的区别是，**静态代理的的代理类是我们自己定义好的，在程序运行之前就已经编译完成，**但是动态代理的代理类是在程序运行时创建的。相比于静态代理，动态代理的优势在于可以很方便的对代理类的函数进行统一的处理，而不用修改每个代理类中的方法。如果想在每个代理方法前都添加一个处理方法。那就要写很多，但是动态代理可以统一管理，也就是如JDK实现，在invoke的重写函数中统一写掉。那么JDK的动态代理是如何实现的呢，看源码

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);

        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }

        /*
         * Look up or generate the designated proxy class.
         */
        Class<?> cl = getProxyClass0(loader, intfs);

        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }

            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
  }

```
然后，我们需要重点关注Class<?> cl = getProxyClass0(loader, intfs)这句代码，这里产生了代理类，这个类就是动态代理的关键，由于是动态生成的类文件，我们将这个类文件打印到文件中。
```java
byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", Student.class.getInterfaces());
        String path = "/Users/mapei/Desktop/okay/65707.class";

        try{
            FileOutputStream fos = new FileOutputStream(path);
            fos.write(classFile);
            fos.flush();
            System.out.println("代理类class文件写入成功");
        }catch (Exception e) {
            System.out.println("写文件错误");
        }
```
对这个class文件进行反编译，我们看看jdk为我们生成了什么样的内容：

```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import proxy.Person;

public final class $Proxy0 extends Proxy implements Person
{
  private static Method m1;
  private static Method m2;
  private static Method m3;
  private static Method m0;

  /**
  *注意这里是生成代理类的构造方法，方法参数为InvocationHandler类型，看到这，是不是就有点明白
  *为何代理对象调用方法都是执行InvocationHandler中的invoke方法，而InvocationHandler又持有一个
  *被代理对象的实例，就可以去调用真正的对象实例。
  */
  public $Proxy0(InvocationHandler paramInvocationHandler)
    throws
  {
    super(paramInvocationHandler);
  }

  //这个静态块本来是在最后的，我把它拿到前面来，方便描述
   static
  {
    try
    {
      //看看这儿静态块儿里面的住giveTask通过反射得到的名字m3，其他的先不管
      m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[] { Class.forName("java.lang.Object") });
      m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
      m3 = Class.forName("proxy.Person").getMethod("giveTask", new Class[0]);
      m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
      return;
    }
    catch (NoSuchMethodException localNoSuchMethodException)
    {
      throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
    }
    catch (ClassNotFoundException localClassNotFoundException)
    {
      throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
    }
  }

  /**
  *
  *这里调用代理对象的giveTask方法，直接就调用了InvocationHandler中的invoke方法，并把m3传了进去。
  *this.h.invoke(this, m3, null);我们可以对将InvocationHandler看做一个中介类，中介类持有一个被代理对象，在invoke方法中调用了被代理对象的相应方法。通过聚合方式持有被代理对象的引用，把外部对invoke的调用最终都转为对被代理对象的调用。
  */
  public final void giveTask()
    throws
  {
    try
    {
      this.h.invoke(this, m3, null);
      return;
    }
    catch (Error|RuntimeException localError)
    {
      throw localError;
    }
    catch (Throwable localThrowable)
    {
      throw new UndeclaredThrowableException(localThrowable);
    }
  }
}

```
再来看看AOP的源码分析

1、看一下bean如何被包装为proxy

```java
protected Object createProxy(
   		Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

   	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
   		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   	}

       // 1.创建proxyFactory，proxy的生产主要就是在proxyFactory做的
   	ProxyFactory proxyFactory = new ProxyFactory();
   	proxyFactory.copyFrom(this);

   	if (!proxyFactory.isProxyTargetClass()) {
   		if (shouldProxyTargetClass(beanClass, beanName)) {
   			proxyFactory.setProxyTargetClass(true);
   		}
   		else {
   			evaluateProxyInterfaces(beanClass, proxyFactory);
   		}
   	}

       // 2.将当前bean适合的advice，重新封装下，封装为Advisor类，然后添加到ProxyFactory中
   	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   	for (Advisor advisor : advisors) {
   		proxyFactory.addAdvisor(advisor);
   	}

   	proxyFactory.setTargetSource(targetSource);
   	customizeProxyFactory(proxyFactory);

   	proxyFactory.setFrozen(this.freezeProxy);
   	if (advisorsPreFiltered()) {
   		proxyFactory.setPreFiltered(true);
   	}

       // 3.调用getProxy获取bean对应的proxy
   	return proxyFactory.getProxy(getProxyClassLoader());
   }

```
2、创建何种类型的Proxy？JDKProxy还是CGLIBProxy？

```java
public Object getProxy(ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}
    // createAopProxy()方法就是决定究竟创建何种类型的proxy
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
        // 关键方法createAopProxy()
		return getAopProxyFactory().createAopProxy(this);
	}

    // createAopProxy()
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
        // 1.config.isOptimize()是否使用优化的代理策略，目前使用与CGLIB
        // config.isProxyTargetClass() 是否目标类本身被代理而不是目标类的接口
        // hasNoUserSuppliedProxyInterfaces()是否存在代理接口
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}

            // 2.如果目标类是接口类（目标对象实现了接口），则直接使用JDKproxy
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}

            // 3.其他情况则使用CGLIBproxy
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
```
3、getProxy()方法

```java
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable// JdkDynamicAopProxy类结构，由此可知，其实现了InvocationHandler，则必定有invoke方法，来被调用，也就是用户调用bean相关方法时，此invoke()被真正调用
   // getProxy()
   public Object getProxy(ClassLoader classLoader) {
   	if (logger.isDebugEnabled()) {
   		logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
   	}
   	Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);

       // JDK proxy 动态代理的标准用法
   	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
   }
```
4、invoke()方法

```java
//使用了JDK动态代理模式，真正的方法执行在invoke()方法里，看到这里在想一下上面动态代理的例子，是不是就完全明白Spring源码实现动态代理的原理了。
			public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		TargetSource targetSource = this.advised.targetSource;
		Class<?> targetClass = null;
		Object target = null;

		try {
            // 1.以下的几个判断，主要是为了判断method是否为equals、hashCode等Object的方法
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				// The target does not implement the equals(Object) method itself.
				return equals(args[0]);
			}
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				// The target does not implement the hashCode() method itself.
				return hashCode();
			}
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}

			Object retVal;

			if (this.advised.exposeProxy) {
				// Make invocation available if necessary.
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			// May be null. Get as late as possible to minimize the time we "own" the target,
			// in case it comes from a pool.
			target = targetSource.getTarget();
			if (target != null) {
				targetClass = target.getClass();
			}
			// 2.获取当前bean被拦截方法链表
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

			// 3.如果为空，则直接调用target的method
			if (chain.isEmpty()) {
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
            // 4.不为空，则逐一调用chain中的每一个拦截方法的proceed，这里的一系列执行的原因以及proceed执行的内容，我 在这里就不详细讲了，大家感兴趣可以自己去研读哈
			else {
				// We need to create a method invocation...
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// Proceed to the joinpoint through the interceptor chain.
				retVal = invocation.proceed();
			}

			...
			return retVal;
		}
	}
	}

```


#### **复合模式**

**复合模式**为另一个对象提供一个替身或占位符以控制对这个对象的访问

1. 复合模式，把一群模式结合起来使用，解决一般性问题
2. MVC是设计模式组合的最经典的案例
3. 模型利用"观察者"让控制器和视图可以随最新的状态改变而更新，视图和控制器实现策略模式，控制器是视图的行为，如果你希望有不同的行为，可以直接换一个控制器，视图内部使用组合模式来管理窗口、按钮以及其他组件
4. 模型对视图和控制器一无所知，也就是解耦的

**案例**

书中为鸭鹅叫的案例，经典的MVC案例

模式总结：

1. 创建型，将客户从所需要实例化的对象中解耦，有Singleton，Builder，Prototype，Abstract Factory，Factory Method
2. 行为型，涉及类和对象如何交互及分配职责，有Visitor，Mediator，Template Method，Command，Iterator，Memento，Interpreter，Observer，State，Strategy，Chain of Responsibility
3. 结构型，把类或对象组合到更大的结构中，有Decorator，Proxy，Composite，Facade，Flyweight，Bridge，Adapter
