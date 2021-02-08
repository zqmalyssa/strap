---
layout: post
title: Java反射总结
tags: [code, java]
author-id: zqmalyssa
---

Java反射是一个需要知道的技术点，很多框架是以此为基础构建

#### JAVA反射

简单的说，反射机制就是在程序的运行过程中被允许对程序本身进行操作，比如自我检查，进行装载，还可以获取类本身，类的所有成员变量和方法，类的对象，还可以在运行过程中动态的创建类的实例，通过实例来调用类的方法，这就是反射机制一个比较重要的功能了。那么要通过程序来理解反射机制，首先要理解类的加载过程。

![reflect]({{ "/assets/img/java/reflect.png" | relative_url}})

在Java程序执行的时候，要经历三个步骤：加载、连接和初始化。首先程序要加载到JVM的方法区中，然后进行连接，最后初始化。这里就主要介绍一下类的加载。如上图，首先，JVM会从硬盘中读取Java源文件并将其加载到方法区中同时生成类名.class文件，也就是类对象，这个类对象中包含了我们创建类的实例时所需要的模板信息，也就是源代码中的成员变量和方法等。Class本身也是一个类，它的主要功能之一就是生成类加载时的class文件，为类的初始化及实例化做准备。而我们在程序中通过关键字new创建的对象创建的是类的对象，而不是类对象，二者的区别如上图所示。

对类的加载有了一个大致的理解之后，我们来看一下实现反射机制的具体操作：

反射机制在我们所学习的框架中有很大的应用，而在我们实际开发中用的并不多，所以理解反射机制对我们学习框架来说很有帮助。

```java

public class User {

    public String name;
    private int age;

    public User(String name, int age) {
      super();
      this.name = name;
      this.age = age;
    }

    private User(int age)
    {
      super();
      this.age = age;

    }

    public User(String name)
    {
      super();
      this.name = name;
    }

    public User() {
      super();
      // TODO Auto-generated constructor stub
    }
    @Override
    public String toString() {
      return "User [name=" + name + ", age=" + age + "]";
    }

    public void exit()
    {
      System.out.println(name+"退出系统");
    }

    public void login(String username,String password)
    {
      System.out.println("用户名:"+username);
      System.out.println("密码:"+password);
    }

    private String CheckInfo()
    {
      return "年龄:"+age;
    }

}

```

利用反射机制可以获取类对象（也就是我们前面介绍的类对象，获取类对象之后我们便获取了类的模板，可以对类进行一些操作），有以下三种方法：
1.类名.class()
2.对象名.getClass()
3.Class.forName(具体的类名)

我们通过代码来看一下具体的操作：


```java
public class Demo {

  public static void main(String args[]) {
    //1.类名.class， 字节码

    Class clz = User.class;
    System.out.println(clz);

    //2.对象名.getClass()， 创建对象

    Class clz1 = new User().getClass();
    System.out.println(clz==clz1);

    //3.Class.forName()，源码

    //Class clz2 = Class.forName("User");
    Class clz2 = null;
    try {
      clz2 = Class.forName("com.qiming.test.reflect.User");
    } catch (ClassNotFoundException e) {
      e.printStackTrace();
    }
    System.out.println(clz==clz2);

  }

}

```

第一行就是所获取的类对象，下面两行true的结果表示类对象只有一个(这里要注意的是第三种方法中类名一定要写全称，包名也要包括进去，不然JVM会无法定位这个类的具体位置)。三种方法分别代表类加载的三个阶段 源码->字节码->创建对象，注释中写出

在类加载的三个阶段里都可以获取类对象，其中在源码中获取类对象是最常用的，也是反射机制在框架中的应用，鉴于我目前的理解，在框架中的应用可以是通过配置文件写入所创建的类名，再利用源码获取方法获取类对象。获取类对象之后就可以对类进行一些创建对象、调用方法、访问成员变量的操作了。创建一个对象

```java
try {
      clz2 = Class.forName("com.qiming.test.reflect.User");
      Object obj = clz2.newInstance();
      System.out.println(obj);
    } catch (ClassNotFoundException e) {
      e.printStackTrace();
    } catch (InstantiationException e) {
      e.printStackTrace();
    } catch (IllegalAccessException e) {
      e.printStackTrace();
    }
```

调用方法：

Method md = 类对象.getMethod("类中的公有方法名");
获取公有方法，其中md是Method类型的方法名，

Object obj1 = md.invoke(u2,"老赵","aixiaoba");
为获取到的方法命名，方便调用。

Method dm = 类对象.getDeclaredMethod("类中的私有方法名");
获取私有方法名，又叫暴力获取，此方法无视方法的访问权限，即使是被private修饰的方法也会被获取到。

Method lm = 类对象.getMethod("有参方法方法名",参数的类对象...);
获取有参方法，同时要获取参数的类对象，格式为：参数类型.class

Object obj1 = lm.invoke(类的对象,"参数1","参数2");
调用获取到的方法，使用invoke关键字，此处表示调用有参方法。

以上方法中由于不确定获取到的对象类型，所以用Object接收。

```java

User u1 = new User("张三", 10);
User u2 = new User("李四", 20);

Method m1 = clz2.getMethod("exit");
Object obj1 = m1.invoke(u2);
System.out.println(obj1); //Object输出为null，因为方法返回值是null
Method m2 = clz2.getMethod("login", String.class, String.class);
Object obj2 = m2.invoke(u2, "赵五", "1234");
System.out.println(obj2); //Object输出为null，因为方法返回值是null
//暴力获取私有方法
Method m3 = clz2.getDeclaredMethod("CheckInfo");
//修改了访问控制级别，因为类中是private修饰
m3.setAccessible(true);
Object obj3 = m3.invoke(u1);
System.out.println(obj3); //Object输出有值，因为有返回值

```

访问成员变量：

最重要的一个关键字：Field

Field nf = 类对象.getField("成员变量名");
和调用私有方法一样，要访问私有成员变量也要通过暴力获取的的方式，同时也要获取访问私有成员变量的权限。

Field af = 类对象.getDeclaredField("私有成员变量名");
af.setAccessible(true);

```java

Object newInstance = clz2.newInstance();
Field f1 = clz2.getField("name");
f1.set(newInstance, "钱六");
Object obj4 = f1.get(newInstance);  //这个object是什么呢，输出就是 钱六
System.out.println(obj4);
System.out.println(newInstance); //这是User对象

User test = new User("王八", 40);
Field f2 = clz2.getDeclaredField("age");
f2.setAccessible(true);
Object obj5 = f2.get(test);
System.out.println(obj5);

```

#### JAVA反射使用注意

1、如果反射的类有@Service，@Component等注解，即被Spring托管，那么不能直接去new一个instance的，解决方法是

```java
method.invoke(GetBeanUtil.getBean(beanName), Param);

public class GetBeanUtil implements ApplicationContextAware {
  protected static ApplicationContext applicationContext;

  @Override
  public void setApplicationContext(ApplicationContext arg0) throws BeansException {
    if (applicationContext == null) {
      applicationContext = arg0;
    }
  }

  public static Object getBean(String name) {
    return applicationContext.getBean(name);
  }

  public static <T> T getBean(Class<T> clazz) {
    return applicationContext.getBean(clazz);
  }
}
```
