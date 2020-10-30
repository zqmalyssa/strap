---
layout: post
title: Java驱动Sh脚本
tags: [code, java, linux]
author-id: zqmalyssa
---

接的第一个项目，通过X通道去执行脚本，那么美

### java如何去驱动Sh脚本

主要注意的地方就是脚本的权限问题，在java中先将没有x权限的脚本赋予权限后再去运行

下面是一个测试类，可以放到linux服务器上进行测试

```java
package com.qiming.test;

import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.util.Arrays;

public class TestSh {

  public static void main(String args[]) {
//    StringBuffer paras = new StringBuffer();
//    Arrays.stream(para).forEach(x -> paras.append(x).append(" "));
    try {
      boolean execCmd = false;
      String cmd = "", shpath = "";
      if (execCmd) {
        // 命令模式
        shpath = "echo";
      } else {
        //脚本路径
        shpath = "/root/test.sh";

      }
//      cmd = shpath + " " + paras.toString();
      cmd = shpath;

      //解决脚本没有执行权限
      ProcessBuilder builder = new ProcessBuilder("/bin/chmod", "755", shpath);
      Process process = builder.start();
      process.waitFor();

      Process ps = Runtime.getRuntime().exec(cmd);
      ps.waitFor();

      BufferedReader br = new BufferedReader(new InputStreamReader(ps.getInputStream()));
      StringBuffer sb = new StringBuffer();
      String line;
      while ((line = br.readLine()) != null) {
        sb.append(line).append("\n");
      }
      String result = sb.toString();
      System.out.println(result);

    } catch (
        Exception e) {
      e.printStackTrace();
    }
  }
}
```

可以看出程序的打印输出，shell脚本内的内容自己替换，还有一块是加传参的

此方法是驱动部署在本机上进行shell命令，如果要驱动到别的机器上呢，那还是得有agent，如saltstack或者ansible
