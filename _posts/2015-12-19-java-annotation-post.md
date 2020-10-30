---
layout: post
title: JAVA的注解
tags: [code, java]
author-id: zqmalyssa
---

注解的基本概念和使用

#### 基本概述

注解，也被称为元数据，为我们在代码中添加信息提供了一种形式化的方法，使我们可以在稍后某个时刻非常方便的使用这些数据，注解在一定程度上是把元数据和源代码文件结合在一起，而不是保存在外部文档中这一大的趋势之下所催生的

定义在`java.lang`中的注解

1. @override，表示当前的方法定义会覆盖超类中的方法，如果你不小心拼写错误，编译器就会发出提示
2. @Deprecated，编译器会发出告警信息
3. @SuppressWarnings，关闭不当的编译器警告信息

还有四种元注解，元注解专职负责注解其他注解

1. @Target，表述注解可以用在什么地方
  - CONSTRUCTOR，构造器的声明
  - FIELD，域声明(包含ENUM实例)
  - LOCAL_VARIABLE，局部变量声明
  - METHOD，方法声明
  - PACKAGE，包声明
  - PARAMETER，参数声明
  - TYPE，类、接口(包括注解类型)或enum声明
2. @Retention，表示需要在什么级别保存该注解信息
  - SOURCE，注解将被编译器丢弃
  - CLASS，注解在class文件中可用，但会被VM丢弃
  - RUNTIME，VM将在运行期也保留注解，因此可以通过反射机制读取注解的信息
3. @Documented，将此注解包含在Javadoc中
4. @Inherited，允许子类继承父类的注解

注解元素可以用的类型如下：

1. 所有基本类型(int、float、boolean等)
2. String
3. Class
4. enum
5. Annotation
6. 以上类型的数组

如果使用了其他类型，那么编译器就会报错，也不允许使用任何包装类型，举个例子

定义一个注解

```java
package com.qiming.test.annoation;


import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 定义一个注解
 */

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface UseCase {

  public int id();
  public String description() default "no description";

}

```
实现注解处理器

```java
package com.qiming.test.annoation;

import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * 注解处理器
 */

public class PasswordUtil {

  @UseCase(id = 47, description = "test 1")
  public boolean validatePassword(String password) {
    return true;
  }

  @UseCase(id = 48)
  public String encryptPassword(String password) {
    return "password";
  }

  @UseCase(id = 49, description = "test 3")
  public boolean checkForNewPassword() {
    return false;
  }

  public static void trackUserCases(List<Integer> useCases, Class<?> cl) {
    //第一个反射
    for (Method method : cl.getDeclaredMethods()) {
      //第二个反射，返回指定类型的注解对象，在这里就是UseCase
      UseCase uc = method.getAnnotation(UseCase.class);
      if (uc != null) {
        System.out.println("Found Use Case: " + uc.id() + " " + uc.description() );
        useCases.remove(new Integer(uc.id()));
      }
    }
    for (int i : useCases) {
      System.out.println("Warning: Missing use case-" + i);
    }
  }

  public static void main(String[] args) {
    List<Integer> useCases = new ArrayList<Integer>();
    Collections.addAll(useCases, 47, 48, 49, 50);
    trackUserCases(useCases, PasswordUtil.class);
  }

}

```
这里面就用到了两个反射方法

再看一个自定义的数据库表元数据，这个应该非常熟悉了

```java
package com.qiming.test.annoation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 定义一个table注解
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface DBTable {

  public String name() default "";

}

```

```java
package com.qiming.test.annoation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 定义一个约束注解
 */

//作用在属性上
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Constraints {

  boolean primaryKey() default false;
  boolean allowNull() default true;
  boolean unique() default false;

}

```

```java
package com.qiming.test.annoation;


import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 定义一个String类型
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SQLString {

  int value() default 0;
  String name() default "";
  Constraints constraints() default @Constraints;

}

```

```java
package com.qiming.test.annoation;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * 定义一个整型
 */
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface SQLInteger {

  String name() default "";
  Constraints constraints() default @Constraints;

}

```
定义一个model，用上注解

```java
package com.qiming.test.annoation;

@DBTable(name = "MEMBER")
public class Member {

  @SQLString(30)
  String firstName;

  @SQLString(50)
  String listName;

  @SQLInteger
  Integer age;

  @SQLString(value = 30, constraints = @Constraints(primaryKey = true))
  String handle;

  static int memeberCount;

  public String getFirstName() {
    return firstName;
  }

  public String getListName() {
    return listName;
  }

  public Integer getAge() {
    return age;
  }

  public String getHandle() {
    return handle;
  }

  public String toString() {
    return handle;
  }
}

```
实现相应的处理器

```java
package com.qiming.test.annoation;

import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.util.ArrayList;
import java.util.List;

/**
 * 自定义Table的处理器
 */
public class TableCreator {

  public static void main(String[] args) throws Exception{

    if (args.length < 1) {
      System.out.println("arguments: annotated classes");
      System.exit(0);
    }

    for (String className : args) {
      //要是类的全路径
      Class<?> cl = Class.forName(className);
      DBTable dbTable = cl.getAnnotation(DBTable.class);
      if (dbTable == null) {
        System.out.println("No DBTable annotation in class " + className);
        continue;
      }

      String tableName = dbTable.name();
      //如果名字为空，则用class的名字
      if (tableName.length() < 1) {
        tableName = cl.getName().toUpperCase();
      }

      List<String> columnDefs = new ArrayList<String>();
      for (Field field : cl.getDeclaredFields()) {
        String columnName = null;
        Annotation[] anns = field.getDeclaredAnnotations();
        if (anns.length < 1) {
          continue;
        }
        if (anns[0] instanceof SQLInteger) {
          SQLInteger sInt = (SQLInteger) anns[0];
          //如果名字为空，就用属性名
          if (sInt.name().length() < 1) {
            columnName = field.getName().toUpperCase();
          } else {
            columnName = sInt.name();
          }
          columnDefs.add(columnName + " INT" + getConstraints(sInt.constraints()));
        }

        if (anns[0] instanceof SQLString) {
          SQLString sString = (SQLString) anns[0];
          if (sString.name().length() < 1) {
            columnName = field.getName().toUpperCase();
          } else {
            columnName = sString.name();
          }
          columnDefs.add(columnName + " VARCHAR(" + sString.value() + ")" + getConstraints(sString.constraints()));
        }
        StringBuilder createCommand = new StringBuilder("CREATE TABLE " + tableName + "(");
        for (String columnDef : columnDefs) {
          createCommand.append("\n     " + columnDef + ",");
        }
        String tableCreate = createCommand.substring(0, createCommand.length() - 1) + ");";
        System.out.println("Table Creation SQL for " + className + " is :\n" + tableCreate);
      }

    }
  }

  private static String getConstraints(Constraints con) {
    String constraints = "";
    if (!con.allowNull()) {
      constraints += " NOT NULL";
    }
    if (con.primaryKey()) {
      constraints += " PRIMARY KEY";
    }
    if (con.unique()) {
      constraints += " UNIQUE";
    }
    return constraints;
  }

}

```

运行的时候，将Member的包名作为输入参数
