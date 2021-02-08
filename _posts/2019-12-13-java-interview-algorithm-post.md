---
layout: post
title: JAVA经典算法题
author-id: "zqmalyssa"
tags: [algorithm, code, java]
---

使用java去写的一些经典算法题，每个自成一体

#### 1.构建二叉搜索树后查找路径

题目：先用数组构建一颗二叉树，然后找出和为某个值得全路径并输出

思路：首先熟悉二叉查找树的构建方法，然后路径中需要用到栈的特性去保存之前的路径

```java
package com.qiming.algorithm;

import java.util.Iterator;
import java.util.LinkedList;
import java.util.Scanner;

/**
 * 先构建二叉搜索树，再找匹配的所有路径并打印出来
 */
public class BinTreePathFind {

  //路径计数器
  private static int count = 0;

  public static void main(String[] args) {
    Scanner sc = new Scanner(System.in);
    //获取第一行输入，默认是一个数值
    int N  = Integer.valueOf(sc.nextLine());
    //获取第二行输入，默认是一串逗号间隔的数字字符串
    String line =  sc.nextLine();
    //开始计算
    new BinTreePathFind().compute(N, line);
  }

/**
 * 构建二叉树搜索树及计算路径的入口
 * @param N 期望值
 * @param line 输入的字符串
 */
  public void compute(int N, String line){
    String[] arr = line.split(",");
    int len = arr.length;
    if (len == 0) {
      throw new InvalidNodeException("输入错误，请输入至少一个元素");
    }
    int[] num = new int[len];
    for (int i = 0; i < len; i++) {
      num[i] = Integer.valueOf(arr[i]);
    }

    BinTreeNode root = buildBST(num);
    findPath(root, N);
  }

/**
 * 用整型数组构建二叉搜索树
 * @param num 整型数组
 * @return
 */
  private BinTreeNode buildBST(int[] num) {
    BinTreeNode root = new BinTreeNode(num[0], null, null);
    for (int i = 1; i < num.length; i++) {
      createBST(root, num[i]);
    }
    return root;
  }

/**
 * 递归构建二叉搜索树
 * @param node
 * @param val
 */
  private void createBST(BinTreeNode node, int val) {
    if (val == node.getVal()) {
      //有值相同，报错
      throw new InvalidNodeException("输入错误，不能有值相同的元素");
    } else if (val < node.getVal()) {
      if (null == node.getLeft()) {
        node.setLeft(new BinTreeNode(val, null, null));
      } else {
        createBST(node.getLeft(), val);
      }
    } else {
      if (null == node.getRight()) {
        node.setRight(new BinTreeNode(val, null, null));
      } else {
        createBST(node.getRight(), val);
      }
    }
  }

/**
 * 查找路径入口
 * @param root
 * @param expectedSum
 */
  private void findPath(BinTreeNode root, int expectedSum) {
    if (root == null) {
      return;
    }
    LinkedList<BinTreeNode> stack = new LinkedList<BinTreeNode>();
    findCorePath(root, expectedSum, stack, 0);
    if (count == 0) {
      System.out.print("error");
    }
  }

/**
 * 查找路径核心方法
 * @param root
 * @param expectedSum
 * @param stack
 * @param currentSum
 */
  private void findCorePath(BinTreeNode root, int expectedSum, LinkedList<BinTreeNode> stack, int currentSum) {
    //入栈
    currentSum += root.getVal();
    stack.push(root);
    if (currentSum == expectedSum && root.isLeaf()) {
      Iterator<BinTreeNode> iterator = stack.descendingIterator();
      StringBuilder sb = new StringBuilder();
      while(iterator.hasNext()) {
        sb.append(iterator.next().getVal()).append(",");
      }
      System.out.print(sb.substring(0, sb.length()-1) + "\n");
      count++;
    }
    //否则继续寻找
    if (root.getLeft() != null) {
      findCorePath(root.getLeft(), expectedSum, stack, currentSum);
    }
    if (root.getRight() != null) {
      findCorePath(root.getRight(), expectedSum, stack, currentSum);
    }
    stack.pop();
  }
}

/**
 * 定义二叉搜索树Node
 */
class BinTreeNode {

  private int val;
  private BinTreeNode left;
  private BinTreeNode right;

  public BinTreeNode(int val, BinTreeNode left, BinTreeNode right) {
    this.val = val;
    this.left = left;
    this.right =right;
  }

  public int getVal() {
    return val;
  }

  public void setVal(int val) {
    this.val = val;
  }

  public BinTreeNode getLeft() {
    return left;
  }

  public void setLeft(BinTreeNode left) {
    this.left = left;
  }

  public BinTreeNode getRight() {
    return right;
  }

  public void setRight(BinTreeNode right) {
    this.right = right;
  }

  public boolean isLeaf() {
    return this.getLeft() == null && this.getRight() == null;
  }

}

/**
 * 定义基本异常
 */
class InvalidNodeException extends RuntimeException {

  public InvalidNodeException(String err) {
    super(err);
  }

}


```

#### 2.斐波那契数列

题目：实现斐波那契数列

思路：可以采用递归的方式，但效率不好，采用由下往上保存值得方式

```java
package com.qiming.algorithm;

/**
 * 斐波那契数列
 */
public class Fibonacci {

  public static void main(String args[]) {
    System.out.println(fibonacci(5));
    System.out.println(fibonacciGood(8));
  }

  /**
   * 这种递归算法重复的计算太多了
   * @param n
   * @return
   */
  private static int fibonacci(int n) {
    if (n <= 0) {
      return 0;
    }
    if (n == 1) {
      return 1;
    }
    return fibonacci(n - 1) + fibonacci(n -2 );
  }

  /**
   * 从下往上计算，先计算0跟1得到f(2)，再计算1跟2得到f(3)，时间复杂度只有O(n)
   * @param n
   * @return
   */
  private static int fibonacciGood(int n) {
    int result[] = {0, 1};
    if (n < 2) {
      return result[n];
    }
    int fibNMinusOne = 1;
    int fibNMinusTwo = 0;
    int fibN = 0;
    for (int i = 2; i <= n; i++) {
      fibN = fibNMinusOne + fibNMinusTwo;
      fibNMinusTwo = fibNMinusOne;
      fibNMinusOne = fibN;
    }
    return fibN;
  }

}

```

#### 3.链表中倒数第K个数

题目：找到单链表中倒数第K个数

思路：让第一个指针先走k-1步，让第二个指针再同步往前走，第一个指针到末尾，第二个自然到倒数K

```java
package com.qiming.algorithm;

/**
 * 找到链表中倒数第K个结点
 * 让第一个指针先走k-1步，第二个指指针跟其始终保持这个距离即可
 */
public class LastKNode {

  public static void main(String args[]) {
    SNode s1 = new SNode(1, null);
    SNode s2 = new SNode(2, s1);
    SNode s3 = new SNode(3, s2);
    SNode s4 = new SNode(4, s3);
    SNode s5 = new SNode(5, s4);
    SNode s6 = new SNode(6, s5);
    SNode s7 = new SNode(7, s6);

    System.out.println(findLastKNode(s7, 3).getVal());
  }

  private static SNode findLastKNode(SNode head, int k) {
    if (head == null || k == 0) {
      return null;
    }
    SNode first = head, second = head;
    for (int i = 0; i < k-1; ++i) {
      if (first.getNext() != null) {
        first = first.getNext();
      } else {
        return null;
      }
    }
    while (first.getNext() != null) {
      first = first.getNext();
      second = second.getNext();
    }
    return second;
  }

}

```

#### 4.计算一个数组中的最大差值

题目：计算一个数组中的最大差值，该题又分左边去减右边和右边去减左边

思路：用min最小值，max_diff差最大值分别进行记录，一次遍历就可以完成

```java
package com.qiming.algorithm;

/**
 * 计算一个数组中的差值最大时，条件是只能左边减右边的数或者只能右边减左边的数，一次遍历就应该能完成
 */
public class MaxDIffOfArray {

  public static void main(String args[]) {

    int[] num1 = {1,4};
//    System.out.println(max_difference_left_to_right(num1));
    System.out.println(max_difference_right_to_left(num1));

  }

  //从左往右减
  private static int max_difference_left_to_right(int[] s) {
    int len = s.length;
    if (len < 2) {
      return 0;
    }
    int min = Math.min(s[len - 1], s[len - 2]);
    int max_diff = s[len - 2] - s[len - 1];
    for (int i = len - 3; i > 0; i--) {
      if (s[i] - min > max_diff) {
        max_diff = s[i] - min;
      }
      if (s[i] < min) {
        min = s[i];
      }
    }
    return max_diff;
  }

  //从右往左减
  private static int max_difference_right_to_left(int[] s) {
    int len = s.length;
    if (len < 2) {
      return 0;
    }
    int min = Math.min(s[0], s[1]);
    int max_diff = s[1] - s[0];

    for (int i = 2; i < len; i++) {
      if (s[i] - min > max_diff) {
        max_diff = s[i] - min;
      }
      if (s[i] < min) {
        min = s[i];
      }
    }
    return max_diff;
  }


}

```
#### 5.求一个int数组中的中位数

题目：求一个int数组中的中位数，中位数就是有序后的中间的数

思路：改造快排中的partition中的算法，加上定位操作，分奇偶数组

```java
package com.qiming.algorithm;

/**
 * 求一个int数组的中位数，中位数是有序的中位数
 * 方法是稍微改造下快排的partition
 */

public class MediumFind {

  public static void main(String args[]) {
//    int[] num1 = {18,14,20,40,11,8,5};
//    int[] num2 = {18,14,20,40,11,8,5,51};
    int[] num1 = {17,15,24,14,3,12,1};
    int[] num2 = {18,14,20,40,11,8,5,51,2,45,21,39,31,16};
    int medium1 = partition(num1, 0, num1.length - 1);
    int medium2 = partition(num2, 0, num2.length - 1);
    System.out.println(medium1);
    System.out.println(medium2);
  }

  //改造一下partition方法
  private static int partition(int s[], int low, int high) {
    //需要加两个变量
    int start = low;
    int end = high;

    int pivot = s[low];
    while (low < high) {
      while (low < high && s[high] >= pivot) {
        high --;
      }
      s[low] = s[high];
      while (low < high && s[low] <= pivot) {
        low ++;
      }
      s[high] = s[low];
    }
    //pivot要被填到坑里面
    s[low] = pivot;
//    return low;

    //以下是定位部分
    if (low == (s.length -1 )/2) {
      return s[low];
    } else if (low > (s.length -1 )/2) {
      return partition(s, start, low-1);
    } else {
      return partition(s, low + 1, end);
    }

  }

}

```

#### 6.倒序打印链表

题目：倒序的去打印一个单链表

思路：用栈存储数据并打印

```java
package com.qiming.algorithm;


import com.qiming.test.datastructureAndAlgorithm.common.StackEmptyException;
import java.util.Iterator;
import java.util.LinkedList;

/**
 * 倒序打印链表
 */
public class PrintLinkedList {

  public static void main(String args[]) {

    SNode s1 = new SNode(1, null);
    SNode s2 = new SNode(2, s1);
    SNode s3 = new SNode(3, s2);
    SNode s4 = new SNode(4, s3);
    SNode s5 = new SNode(5, s4);
    SNode s6 = new SNode(6, s5);
    SNode s7 = new SNode(7, s6);

    SNode head = new SNode(-1, s7);

//    while(head.getNext() != null) {
//      System.out.println(head.getNext().getVal());
//      head = head.getNext();
//    }

//    new PrintLinkedList().printReversingly(head);
    new PrintLinkedList().printReversinglyBySelfStack(head);

  }

  private void printReversingly(SNode head) {

    LinkedList<SNode> stack = new LinkedList();

    while(head.getNext() != null) {
      stack.push(head.getNext());
      head = head.getNext();
    }

    Iterator iterator = stack.iterator();
    while(iterator.hasNext()) {
      SNode s = (SNode)iterator.next();
      System.out.println(s.getVal());
    }

  }

  private void printReversinglyBySelfStack(SNode head) {

    Stack stack = new Stack();
    while(head.getNext() != null) {
      stack.push(head.getNext());
      head = head.getNext();
    }

    while(!stack.isEmpty()) {
      System.out.println(stack.pop().getVal());
    }

  }


}

/**
 * Node设计
 */
class SNode {

  private int val;
  private SNode next;

  public SNode(int val, SNode next){
    this.val = val;
    this.next = next;
  }

  public int getVal() {
    return val;
  }

  public void setVal(int val) {
    this.val = val;
  }

  public SNode getNext() {
    return next;
  }

  public void setNext(SNode next) {
    this.next = next;
  }
}

/**
 * 栈设计
 */
class Stack {
  private SNode top;
  private int size;

  public Stack() {
    this.top = null;
    this.size = 0;
  }

  public int getSize() {return size;};

  public boolean isEmpty() {return size == 0;};

  public void push(SNode node) {
    SNode p = new SNode(node.getVal(), top);
    top = p;
    size++;
  }

  public SNode pop() {
    if (size < 1) {
      throw new StackEmptyException("错误，堆栈为空。");
    }
    SNode result = top;
    top = top.getNext();
    size--;
    return result;
  }

  public SNode peek() {
    if (size < 1) {
      throw new StackEmptyException("错误，堆栈为空。");
    }
    return top;
  }
}
```

#### 7.奇数全部放到偶数的前面

题目：对一个int数组，将奇数全部放到偶数的前面

思路：可以使用两个指针，头部向后直到遇到偶数，尾部向前直到遇到奇数，然后交换

```java
package com.qiming.algorithm;

/**
 * 奇数全部放到偶数的前面
 * 可以使用两个指针，头部向后直到遇到偶数，尾部向前直到遇到奇数
 */
public class PutOddBeforeEven {

  public static void main(String args[]) {

    int s[] = {1,2,3,4,5,6,7,8,9};
    changeArray(s);
    for (int i = 0; i < s.length; i++) {
      System.out.println(s[i]);
    }

  }

  private static void changeArray(int s[]) {

    if (s == null || s.length <= 1) {
      return;
    }
    int begin = 0;
    int end = s.length - 1;

    while (begin < end) {
      //向后移动begin，直到它遇到偶数
      while (begin < end && (s[begin] & 1) != 0) {
        begin++;
      }

      //向前移动end，直到它遇到奇数
      while (begin < end && (s[end] & 1) == 0) {
        end--;
      }

      if (begin < end) {
        int temp = s[begin];
        s[begin] = s[end];
        s[end] = temp;
      }
    }

  }

}

```

#### 8.用两个栈实现队列的操作

题目：用两个栈实现一个队列，完成队列的appendTail和deleteHead方法

思路：第一个栈用来接刚插入的元素，只要有删除，只要stack2中有数据，就删一个栈顶元素，否则就从stack1中将数据弹入stack2

```java
package com.qiming.algorithm;

/**
 * 用两个栈实现一个队列，完成队列的appendTail和deleteHead方法
 * 第一个栈用来接刚插入的元素，只要有删除，只要stack2中有数据，就删一个栈顶元素，否则就从stack1中将数据弹入stack2
 */

import com.qiming.test.datastructureAndAlgorithm.common.StackEmptyException;

public class TwoStackForAQueue {

  //两个栈，要初始化
  private Stack stack1 = new Stack();
  private Stack stack2 = new Stack();

  public void appendTail(SNode s) {
    stack1.push(s);
  }

  public SNode deleteHead() {
    //删前先从stack1中搞过来
    if (stack2.getSize() <= 0) {
      while(stack1.getSize() > 0) {
        SNode p = stack1.peek();
        stack1.pop();
        stack2.push(p);
      }
    }
    //如果还是0的话
    if (stack2.getSize() == 0) {
      throw new StackEmptyException("Queue is empty");
    }

    //正常是有值得，进行删除
    SNode head = stack2.pop();
    return head;
  }

  public static void main(String args[]) {
    SNode s1 = new SNode(1, null);
    SNode s2 = new SNode(2, null);
    SNode s3 = new SNode(3, null);
    SNode s4 = new SNode(4, null);
    SNode s5 = new SNode(5, null);
    SNode s6 = new SNode(6, null);
    SNode s7 = new SNode(7, null);

    TwoStackForAQueue twoStackForAQueue = new TwoStackForAQueue();

    twoStackForAQueue.appendTail(s1);
    twoStackForAQueue.appendTail(s2);
    System.out.println(twoStackForAQueue.deleteHead().getVal());
    twoStackForAQueue.appendTail(s3);
    twoStackForAQueue.appendTail(s4);
    System.out.println(twoStackForAQueue.deleteHead().getVal());
    twoStackForAQueue.appendTail(s5);
    System.out.println(twoStackForAQueue.deleteHead().getVal());
    twoStackForAQueue.appendTail(s6);
    twoStackForAQueue.appendTail(s7);
    System.out.println(twoStackForAQueue.deleteHead().getVal());
  }

}


```

#### 9.围成圈的100人报数M，知道剩余M-1人，求剩下人的原数值

题目：围成圈的100人报数M，知道剩余M-1人，求剩下人的原数值

思路：构成100人的链表

```java
package com.qiming.HW;

import java.util.Scanner;

public class FindNum {

  public static void main(String args[]) {

    Scanner in = new Scanner(System.in);
    while (in.hasNextInt()) {// 注意，如果输入是多个测试用例，请通过while循环处理多个测试用例
      int m = in.nextInt();
      if (m <= 1 || m >= 100) {
        System.out.println("ERROR!");
        continue;
      }
      //构建一个循序链表，从1到100
      SNode head = new SNode(1, null);
      SNode start = head;
      for (int i = 2; i <= 100; i++) {
        SNode p = new SNode(i, null);
        head.setNext(p);
        head = head.getNext();
      }
      head.setNext(start);

      SNode newHead = findnum(m, start);
      //打印输出
      StringBuilder sb = new StringBuilder();
      int statrVal = newHead.getVal();
      sb.append(statrVal).append(",");
      while(newHead.getNext().getVal() != statrVal) {
        sb.append(newHead.getNext().getVal()).append(",");
        newHead = newHead.getNext();
      }
      System.out.println(sb.substring(0, sb.length() - 1));
    }
  }

  private static SNode findnum(int m, SNode head) {
    int count = 0;
    int leavePerson = 100 - m + 1;
    SNode tmp = head;
    while (count < leavePerson) {
      for (int i = 0; i < m - 1; i++) {
        head = head.getNext();
      }
      for (int i = 0; i < m - 2; i++) {
        tmp = tmp.getNext();
      }
      //断开链表中的结点
      tmp.setNext(head.getNext());
      SNode tmp2 = head.getNext();
      head = tmp2;
      tmp = tmp2;
      count++;
    }
    return head;
  }

}


class SNode {

  private int val;
  private SNode next;

  public SNode (int val, SNode next) {
    this.val = val;
    this.next = next;
  }

  public int getVal() {
    return val;
  }

  public void setVal(int val) {
    this.val = val;
  }

  public SNode getNext() {
    return next;
  }

  public void setNext(SNode next) {
    this.next = next;
  }

}
```

#### 10.反转一个链表，输出头结点

题目：反转一个链表，输出反转后的头结点

思路：用到三个指针，分别用到前一个，本身和后一个结点保存结点，另外有个反转的结点头

```java
package com.qiming.algorithm;

/**
 * 反转一个链表，输出反转后的头结点
 * 思路就是用到三个指针，分别用到前一个，本身和后一个结点
 */
public class ReverseList {

  public static void main(String args[]) {
    SNode s1 = new SNode(1, null);
    SNode s2 = new SNode(2, s1);
    SNode s3 = new SNode(3, s2);
    SNode s4 = new SNode(4, s3);
    SNode s5 = new SNode(5, s4);
    SNode s6 = new SNode(6, s5);
    SNode s7 = new SNode(7, s6);

    SNode reversed = ReverseList(s7);

    do {
      System.out.println(reversed.getVal());
      reversed = reversed.getNext();
    } while (reversed.getNext() != null);
    //最后再打印一次吧
    System.out.println(reversed.getVal());
  }

  private static SNode ReverseList(SNode head) {
    SNode pReversedHead = null;
    SNode pNode = head;
    SNode pPrev = null;

    while (pNode != null) {
      SNode pnext = pNode.getNext();
      if (pnext == null) {
        pReversedHead = pNode;
      }

      pNode.setNext(pPrev);
      pPrev = pNode;
      pNode = pnext;
    }

    return pReversedHead;
  }

}

```

#### 11.合并两个有序的链表，输出头结点

题目：合并两个有序的链表，输出头结点

思路：比较两个链表的第一个元素，但是要做到鲁棒，记得null的判断，然后用递归写

```java
package com.qiming.algorithm;

/**
 * 前提是两个有序的链表，合并之后还是有序的
 * 比较两个链表的第一个元素，但是要做到鲁棒，记得null的判断，使用递归写
 */
public class MergeTwoList {

  public static void main(String args[]) {
    SNode s1 = new SNode(8, null);
    SNode s2 = new SNode(6, s1);
    SNode s3 = new SNode(4, s2);
    SNode s4 = new SNode(2, s3);

    SNode s5 = new SNode(7, null);
    SNode s6 = new SNode(5, s5);
    SNode s7 = new SNode(3, s6);
    SNode s8 = new SNode(1, s7);

    SNode newHead = mergeTwoList(s4, s8);

    while(newHead != null) {
      System.out.println(newHead.getVal());
      newHead = newHead.getNext();
    }

  }

  private static SNode mergeTwoList(SNode head1, SNode head2) {
    //判断空情况
    if (head1 == null) {
      return head2;
    }
    if (head2 == null) {
      return head1;
    }
    SNode newHead;
    if (head1.getVal() < head2.getVal()) {
      newHead = head1;
      newHead.setNext(mergeTwoList(head1.getNext(), head2));
    } else {
      newHead = head2;
      newHead.setNext(mergeTwoList(head1, head2.getNext()));
    }
    return newHead;
  }

  private static SNode mergeTwoListByIteration(SNode head1, SNode head2) {
    //判断空情况
    if (head1 == null) {
      return head2;
    }
    if (head2 == null) {
      return head1;
    }
    SNode head = new SNode(-1, null);
    SNode result = head;

    //这边是且
    while (head1 != null && head2 != null) {
      if (head1.getVal() > head2.getVal()) {
        head.setNext(head2);
        head = head.getNext();
        head2 = head2.getNext();
      } else if (head1.getVal() < head2.getVal()) {
        head.setNext(head1);
        head = head.getNext();
        head1 = head1.getNext();
      } else {
        SNode s1 = new SNode(head1.getVal(), null);
        SNode s2 = new SNode(head1.getVal(), null);
        head.setNext(s1);
        head = head.getNext();
        head.setNext(s2);
        head = head.getNext();
//        这样写会有问题，1,2步后 head和head1指针就相同了，如果head.setNext(head2);后就会导致head1也发生变化，重复数据用上面的方法搞吧
//        head.setNext(head1);
//        head = head.getNext();
//        head.setNext(head2);
//        head = head.getNext();
        head1 = head1.getNext();
        head2 = head2.getNext();
      }
    }

    //要进行补充
    if (head1 != null) {
      head.setNext(head1);
    }

    if (head2 != null) {
      head.setNext(head2);
    }

    return result.getNext();

}

```

#### 12.输入A，B两个二叉树，判断B是不是A的子结构

题目：输入A，B两个二叉树，判断B是不是A的子结构

思路：分成两步走，第一步找到树A中与B的根结点的值一样的结点R，第二步再判断树A中以R为根结点的子树是不是包含和树B一样的结构，第一步中的值得找法可以用递归遍历，第二步中也是递归

```java
package com.qiming.algorithm;

/**
 * 输入A，B两个二叉树，判断B是不是A的子结构
 * 分成两步走，第一步找到树A中与B的根结点的值一样的结点R，第二步再判断树A中以R为根结点的子树是不是包含和树B一样的结构
 * 遍历可以用递归的方式去遍历，也可以用循环的方式去遍历，面试可以通常用递归的方式进行
 */
public class DoesTree1HasTree2 {

  public static void main(String args[]) {
    BinTreeNode b1 = new BinTreeNode(4, null, null);
    BinTreeNode b2 = new BinTreeNode(7, null, null);

    BinTreeNode b3 = new BinTreeNode(2, b1, b2);
    BinTreeNode b4 = new BinTreeNode(9, null, null);

    BinTreeNode b5 = new BinTreeNode(8, b4, b3);
    BinTreeNode b6 = new BinTreeNode(7, null, null);

    BinTreeNode b7 = new BinTreeNode(8, b5, b6);

    BinTreeNode b8 = new BinTreeNode(9, null, null);
//    BinTreeNode b9 = new BinTreeNode(2, null, null);
    BinTreeNode b9 = new BinTreeNode(3, null, null);
    BinTreeNode b10 = new BinTreeNode(8, b8, b9);

    boolean result = hasSubtree(b7, b10);

    System.out.println(result);

  }

  private static boolean hasSubtree(BinTreeNode root1, BinTreeNode root2) {
    boolean result = false;
    if (root1 != null && root2 != null) {
      if (root1.getVal() == root2.getVal()) {
        result = doesTree1HasTree2(root1, root2);
      }
      //去左子树找找
      if (!result) {
        result = hasSubtree(root1.getLeft(), root2);
      }
      //去右子树找找
      if (!result) {
        result = hasSubtree(root1.getRight(), root2);
      }
    }
    return result;
  }

  private static boolean doesTree1HasTree2(BinTreeNode tree1, BinTreeNode tree2) {

    if (tree2 == null) {
      return true;
    }

    if (tree1 == null) {
      return false;
    }

    if (tree1.getVal() != tree2.getVal()) {
      return false;
    }

    return doesTree1HasTree2(tree1.getLeft(), tree2.getLeft()) && doesTree1HasTree2(tree1.getRight(), tree2.getRight());
  }

}

```

#### 13.二叉树的镜像

题目：输入一个二叉树，输出二叉树的镜像

思路：画图会发现子树是不会离开的，变化位置还是该结点的子树，所以递归去写，代码里面还要树的层次遍历算法

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.concurrent.LinkedBlockingQueue;

/**
 * 输入一个二叉树，函数输出它的镜像
 *
 */
public class BinTreeImage {

  public static void main(String args[]) {

    BinTreeNode b1 = new BinTreeNode(5, null, null);
    BinTreeNode b2 = new BinTreeNode(7, null, null);
    BinTreeNode b3 = new BinTreeNode(6, b1, b2);
    BinTreeNode b4 = new BinTreeNode(9, null, null);
    BinTreeNode b5 = new BinTreeNode(11, null, null);
    BinTreeNode b6 = new BinTreeNode(10, b4, b5);

    BinTreeNode b7 = new BinTreeNode(8, b3, b6);

    //层次遍历
    LinkedList listOld = levelOrder(b7);

    generateImage(b7);

    LinkedList list = levelOrder(b7);

    for (int i = 0; i < list.size(); i++) {
      System.out.println(((BinTreeNode)listOld.get(i)).getVal());
    }

    for (int i = 0; i < list.size(); i++) {
      System.out.println(((BinTreeNode)list.get(i)).getVal());
    }



  }

  private static void generateImage(BinTreeNode root) {
    if (root == null || (root.getLeft() == null && root.getRight() == null)) {
      return;
    }

    BinTreeNode tmp = root.getLeft();
    root.setLeft(root.getRight());
    root.setRight(tmp);

    if (root.getLeft() != null) {
      generateImage(root.getLeft());
    }

    if (root.getRight() != null) {
      generateImage(root.getRight());
    }
  }

  private static LinkedList levelOrder(BinTreeNode root) {
    LinkedList list = new LinkedList();
    if (root == null) {
      return list;
    }
    LinkedBlockingQueue q = new LinkedBlockingQueue();
    try {
      //put和take相对
      q.put(root);
      while (!q.isEmpty()) {
        BinTreeNode p = (BinTreeNode)q.take();
        list.addLast(p);
        if (p.getLeft() != null) {
          q.put(p.getLeft());
        }
        if (p.getRight() != null) {
          q.put(p.getRight());
        }
      }
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    return list;
  }

}
```

#### 14.包含min函数的栈

题目：实现一个能够得到栈的最小元素的min函数，在该栈中，调用min、push和pop的时间复杂度都是O(1)

思路：用一个辅助栈。每次压数据的时候也在辅助栈中压入最小值。弹出的时候如果是最小值，也在辅助栈中弹出

```java
package com.qiming.algorithm;

import java.util.Stack;

/**
 * 包含min函数的栈
 * 实现一个能够得到栈的最小元素的min函数，在该栈中，调用min、push和pop的时间复杂度都是O(1)
 * 思路是用一个辅助栈。每次压数据的时候也在辅助栈中压入最小值。弹出的时候如果是最小值，也在辅助栈中弹出。画画图，注意提交的时候不要用静态变量
 */

public class MinStack {

  private static Stack stack_data = new Stack();
  private static Stack stack_min = new Stack();

  public static void main(String[] args) {
    MinStack test = new MinStack();

    test.push(3);
    System.out.println(test.min());
    test.push(4);
    System.out.println(test.min());
    test.push(2);
    System.out.println(test.min());
    test.push(1);
    System.out.println(test.min());
    test.pop();
    System.out.println(test.min());
    test.pop();
    System.out.println(test.min());
    test.push(0);
    System.out.println(test.min());

  }

  private static int pop (){
    Integer top = null;
    if (!stack_data.isEmpty() && !stack_min.isEmpty()) {
      top = (Integer)(stack_data.pop());
      stack_min.pop();
    }
    return top;
  }

  private static void push (int val){
    stack_data.push(val);
    if (stack_min.isEmpty() || val < (Integer)(stack_min.peek())) {
      stack_min.push(val);
    } else {
      stack_min.push(stack_min.peek());
    }
  }

  private static int min() {
    Integer min = null;
    if (!stack_data.isEmpty() && !stack_min.isEmpty()) {
      min = (Integer)(stack_min.peek());
    }
    return min;
  }

}

```

#### 15.树的最小深度（NC）

题目：求二叉树的最小深度

思路：未采用树的递归，使用层次遍历，左右子树结点为null的就是最小的深度了，判断好边界条件，也可以使用递归

```java
package com.qiming.algorithm.nowcoder;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 求二叉树的最小深度
 */

public class MinimumDepthOfBinTree {

  public int run(TreeNode root) {

    if (root == null) {
      return 0;
    }

    if (root.left == null && root.right == null) {
      return 1;
    }

    //层次遍历
    int depth = 0;
    Queue<TreeNode> queue = new LinkedList<TreeNode>();
    queue.offer(root);
    while(!queue.isEmpty()) {
      int len = queue.size();
      depth++;
      for (int i = 0; i < len; i++) {
        TreeNode cur = queue.poll();
        if (cur.left == null && cur.right == null) {
          return depth;
        }
        if (cur.left != null) {
          queue.offer(cur.left);
        }
        if (cur.right != null) {
          queue.offer(cur.right);
        }
      }
    }
    return 0;
  }

}


class TreeNode {
  int val;
  TreeNode left;
  TreeNode right;
  TreeNode(int x) { val = x; }
}

// 递归的写法

class Solution {
  public int minDepth(TreeNode root) {
    if(root == null) return 0;
    //这道题递归条件里分为三种情况
    //1.左孩子和有孩子都为空的情况，说明到达了叶子节点，直接返回1即可
    if(root.left == null && root.right == null) return 1;
    //2.如果左孩子和由孩子其中一个为空，那么需要返回比较大的那个孩子的深度        
    int m1 = minDepth(root.left);
    int m2 = minDepth(root.right);
    //这里其中一个节点为空，说明m1和m2有一个必然为0，所以可以返回m1 + m2 + 1;
    if(root.left == null || root.right == null) return m1 + m2 + 1;

    //3.最后一种情况，也就是左右孩子都不为空，返回最小深度+1即可
    return Math.min(m1,m2) + 1;
  }
}

// 如果不加左右的判断条件，出现如下单边树的情况会不好处理，其实深度是3，加了判断会加上真实的树的高度
   1
  1
 1 1

// 可以简化
class Solution {
  public int minDepth(TreeNode root) {
    if(root == null) return 0;
    int m1 = minDepth(root.left);
    int m2 = minDepth(root.right);
    //1.如果左孩子和右孩子有为空的情况，直接返回m1+m2+1
    //2.如果都不为空，返回较小深度+1
    return root.left == null || root.right == null ? m1 + m2 + 1 : Math.min(m1,m2) + 1;
  }
}

// 正常想法，不是太好
class Solution {
  public int minDepth(TreeNode root) {
    if(root == null) {
        return 0;
    }
    if(root.left == null && root.right == null) {
        return 1;
    }
    int ans = Integer.MAX_VALUE;
    if(root.left != null) {
        ans = Math.min(minDepth(root.left), ans);
    }
    if(root.right != null) {
        ans = Math.min(minDepth(root.right), ans);
    }
    return ans + 1;
  }
}

```

#### 16.计算逆波兰式（NC）

题目：计算逆波兰式，运算符仅包含"+"，"-"，"*"，"/"，被操作数可能是整数或其他表达式,["2", "1", "+", "3", "*"] -> ((2 + 1) * 3) -> 9↵  ["4", "13", "5", "/", "+"] -> (4 + (13 / 5)) -> 6

思路：用栈，如果遇到数字就入栈，遇到符号就计算，结果再入栈

```java
package com.qiming.algorithm.nowcoder;

import java.util.Stack;

/**
 *  计算逆波兰式，运算符仅包含"+"，"-"，"*"，"/"，被操作数可能是整数或其他表达式
 *  ["2", "1", "+", "3", "*"] -> ((2 + 1) * 3) -> 9↵  ["4", "13", "5", "/", "+"] -> (4 + (13 / 5)) -> 6
 *  思路，用栈，如果遇到数字就入栈，遇到符号就计算，结果再入栈
 */
public class EvaluateReversePolish {

  public static void main(String[] args) {
    String[] s = {"2","1","+","3","*"};
    System.out.println(new EvaluateReversePolish().evalRPN(s));
  }

  public int evalRPN(String[] tokens) {

    if (tokens == null || tokens.length == 0) {
      return -1;
    }
    if (tokens.length == 1) {
      return Integer.parseInt(tokens[0]);
    }
    Stack stack = new Stack();
    int result = 0;
    for (int i = 0; i < tokens.length; i++) {
      if (tokens[i] != "+" && tokens[i] != "-" && tokens[i] != "*" && tokens[i] != "/") {
        stack.push(Integer.parseInt(tokens[i]));
      } else {
        int operand2 = (Integer)stack.pop();
        int operand1 = (Integer)stack.pop();
        char c = tokens[i].charAt(0);
        result = compute(operand1, operand2, c);
        stack.push(result);
      }
    }
    return result;
  }

  private static int compute(int num1, int num2, char c) {
    switch (c) {
      case '+':
        return num1 + num2;
      case '-':
        return num1 - num2;
      case '*':
        return num1 * num2;
      case '/':
        return num1 / num2;
      default:
        return 0;
    }
  }
}

```

#### 17.给链表排序（NC）

题目：给链表排序，要求时间复杂度O(nlogn)，空间复杂度O(1)

思路：单链表的快速排序（n log n）和单链表的归并排序，而如果不能使用递归的话，需要使用归并中的bottom-to-up算法

```java
package com.qiming.algorithm.nowcoder;

/**
 * 在O(n log n)的时间内使用常数级空间复杂度对链表进行排序。
 * 可以理解成不能两个for循环，也不要使用辅助存储
 * 思路是单链表的快速排序（n log n）和单链表的归并排序
 */
public class SortList {

  public static void main(String[] args) {
    ListNode l1 = new ListNode(2);
    ListNode l2 = new ListNode(5);
    ListNode l3 = new ListNode(10);
    ListNode l4 = new ListNode(25);
    ListNode l5 = new ListNode(13);
    ListNode l6 = new ListNode(7);
    ListNode l7 = new ListNode(48);
    ListNode l8 = new ListNode(31);
    l1.next = l2;
    l2.next = l3;
    l3.next = l4;
    l4.next = l5;
    l5.next = l6;
    l6.next = l7;
    l7.next = l8;
    new SortList().sortListByBTU(l1);
    while(l1 != null) {
      System.out.println(l1.val);
      l1 = l1.next;
    }
  }

  public ListNode sortList(ListNode head) {
    return quickSort(head,null);
  }

  private ListNode quickSort(ListNode head, ListNode end) {

    if (head != end) {
      ListNode p = partion(head, end);
      quickSort(head, p);
      quickSort(p.next, end);
    }
    return head;
  }


  /**
   * 单链表的partion的思路才关键，需要理清楚
   * 将原链表看做是两个链表，左链表和右链表，
   * @param head
   * @param end
   * @return
   */
  private ListNode partion(ListNode head, ListNode end) {
    int pivot = head.val;
    ListNode p1 = head;
    ListNode p2 = head.next;

    //这里也包含了只有一个节点的情况
    while(p2 != end) {
      //如果小于基准值，才做处理，否则一直向右移，p2是会始终到最后的，p1根据P2的值决定自己的动作，最后停留在基准点处，那么右子链表的第一个结点是p1的next
      if (p2.val < pivot) {
        //p1向前走
        p1 = p1.next;
        swap(p1, p2);
      }
      p2 = p2.next;
    }
    swap(head, p1);
    return p1;
  }

  private void swap(ListNode p1, ListNode p2) {
    int tmp = p1.val;
    p1.val = p2.val;
    p2.val = tmp;
  }

  /**
   * 以下是使用归并排序的实现，这边还是用到了递归
   */
  public ListNode sortListByMerge(ListNode head){
    if(head==null || head.next==null){
      return head;
    }
    ListNode mid=findMid(head);
    ListNode right=mid.next;
    //防止内存泄漏
    mid.next=null;
    return merge(sortList(head),sortList(right));
  }
  private ListNode merge(ListNode head1,ListNode head2){
    ListNode dummy = new ListNode(0);
    ListNode cur = dummy;
    while(head1!=null && head2!=null){
      if(head1.val<=head2.val){
        cur.next=head1;
        head1=head1.next;
      }else{
        cur.next=head2;
        head2=head2.next;
      }
      cur=cur.next;
    }
    if(head1!=null){
      cur.next=head1;
    }else if(head2!=null){
      cur.next=head2;
    }
    //相当于有头结点
    return dummy.next;
  }
  private ListNode findMid(ListNode head){
    if(head==null){
      return head;
    }
    ListNode slow=head;
    ListNode fast=head;
    while(fast.next!=null && fast.next.next!=null){
      fast=fast.next.next;
      slow=slow.next;
    }
    return slow;
  }

  /**
   * 上面两个都还是用了递归，下面还有个bottom-to-up的思想，不用递归，完成归并排序
   * 可以解决空间复杂度是O(1)的限制
   * 1. 双路归并，2. cut， 3. dummyHead大法
   */
  private ListNode sortListByBTU(ListNode head) {
    ListNode dummyHead = new ListNode(Integer.MIN_VALUE);
    dummyHead.next = head;
    // 先统计长度f
    ListNode p = dummyHead.next;
    int length = 0;
    while(p != null){
      ++length;
      p = p.next;
    }
    // 循环开始切割和合并，<<是乘以2，所以这里的时间复杂度是O(nlogn)
    // 就是先两个两个的 merge，完成一趟后，再 4个4个的 merge，直到结束
    for(int size = 1; size < length; size <<= 1){
      ListNode cur = dummyHead.next;
      ListNode tail = dummyHead;
      while(cur != null){
        ListNode left = cur;
        ListNode right = cut(cur, size); // 链表切掉size 剩下的返还给right
        cur = cut(right, size); // 链表切掉size 剩下的返还给cur
        //merge和mergeByBTU一样
        tail.next = mergeByBTU(left, right);
        while(tail.next != null){
          tail = tail.next; // 保持最尾端
        }
      }
    }
    return dummyHead.next;
  }

  /**
   * 将链表L切掉前n个节点 并返回后半部分的链表头
   * @param head
   * @param n
   * @return
   */
  public static ListNode cut(ListNode head, int n){
    if(n <= 0) return head;
    ListNode p = head;
    // 往前走n-1步
    while(--n > 0 && p != null){
      p = p.next;
    }
    if(p == null) return null;
    ListNode next = p.next;
    p.next = null;
    return next;
  }

  /**
   * 合并list1和list2，跟上面merge一样，就是双路归并，分成顺序存储和链表存储
   * @param list1
   * @param list2
   * @return
   */
  public static ListNode mergeByBTU(ListNode list1, ListNode list2){
    ListNode dummyHead = new ListNode(Integer.MIN_VALUE), p = dummyHead;
    while(list1 != null && list2 != null){
      if(list1.val < list2.val){
        p.next = list1;
        list1 = list1.next;
      } else{
        p.next = list2;
        list2 = list2.next;
      }
      p = p.next;
    }
    if(list1 == null){
      p.next = list2;
    } else{
      p.next = list1;
    }
    return dummyHead.next;
  }


}

class ListNode {
  int val;
  ListNode next;
  ListNode(int x) {
    val = x;
    next = null;
  }
}
```
#### 18.万万没想到之聪明的编辑（NC）

题目：见代码

思路：快慢指针

```java
package com.qiming.algorithm.nowcoder;

import java.util.Scanner;

/**
 * 1. 三个同样的字母连在一起，一定是拼写错误，去掉一个的就好啦：比如 helllo -> hello
 * 2. 两对一样的字母（AABB型）连在一起，一定是拼写错误，去掉第二对的一个字母就好啦：比如 helloo -> hello
 * 3. 上面的规则优先“从左到右”匹配，即如果是AABBCC，虽然AABB和BBCC都是错误拼写，应该优先考虑修复AABB，结果为AABCC
 * 写出一个校准器
 * 思路是使用双指针，并借助StringBuilder或者StringBuffer，其中一个慢指针，一个快指针，k慢的在并不满足规则1,2时随着快i的增加而增加
 * 当满足规则时停止移动，处理开始
 */

public class AutoCorrect {

  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
//    while(true) { //nowcoder中不需要while(true)，一次性的
    int N = Integer.parseInt(scanner.nextLine());
    String[] s = new String[N];
    for (int i = 0; i < N; i++) {
      s[i] = scanner.nextLine();
      s[i] = autoCorrect(s[i]);
    }
    for (int i = 0; i < N; i++) {
      System.out.println(s[i]);
    }

//    }
  }

  private static String autoCorrect(String source) {
    char[] chars = source.toCharArray();
    StringBuffer sb = new StringBuffer();
    int k = 0;
    for (int i = 0; i < chars.length; i++) {
      chars[k] = chars[i];
      sb.append(chars[k]);
      k++;
      //先满足规则1
      if (k >= 3 && chars[k-3] == chars[k-2] && chars[k-2] == chars[k-1]) {
        sb.deleteCharAt(k-1);
        k--;
      }
      //再满足规则2
      if (k >= 4 && chars[k-4] == chars[k-3] && chars[k-2] == chars[k-1]) {
        sb.deleteCharAt(k-1);
        k--;
      }
    }
    return String.valueOf(sb);
  }

  private static String del(String s, int i) {
    return s.substring(0, i) + s.substring(i+1);
  }
}

```

#### 19.树的四种非递归遍历算法（NC）

题目：写出树的先序，中序，后序，层次遍历算法

思路：不用递归，先序，中序，后序用辅助栈，层次用辅助队列

```java
package com.qiming.algorithm.nowcoder;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.Stack;

/**
 * 二叉树的先序，中序，后序的非递归算法
 *         4
 *      5      7
 *   14   6  10   3
 *
 */

public class TreeTraverseNoRecursive {

  public static void main(String[] args) {
    BinTreeNode b1 = new BinTreeNode(4);
    BinTreeNode b2 = new BinTreeNode(5);
    BinTreeNode b3 = new BinTreeNode(7);
    BinTreeNode b4 = new BinTreeNode(14);
    BinTreeNode b5 = new BinTreeNode(6);
    BinTreeNode b6 = new BinTreeNode(10);
    BinTreeNode b7 = new BinTreeNode(3);
    b1.left = b2;
    b1.right = b3;
    b2.left = b4;
    b2.right = b5;
    b3.left = b6;
    b3.right = b7;

    List result = new LinkedList();
//    new TreeTraverseNoRecursive().preOrderTraverseNoRecursive(b1, result);
//    new TreeTraverseNoRecursive().inOrderTraverseNoRecursive(b1, result);
//    new TreeTraverseNoRecursive().postOrderTraverseNoRecursive(b1, result);
    new TreeTraverseNoRecursive().levelOrderTraverseNoRecursive(b1, result);
    for (int i = 0; i < result.size(); i++) {
      System.out.println(((BinTreeNode)result.get(i)).val);
    }
  }

  /**
   * 树的先序遍历
   * 内层循环中，沿着根结点P一直往左走，沿途访问经过的根结点，并将这些根结点的非空右子树入栈，直到p为空
   * 此时就应当取出沿途遇到的最后的非空右子树的根，即栈顶结点，然后在外层循环中继续先序遍历这棵以p指向的子树
   * 如果栈空，表示再没有右子树需要遍历了，外层循环结束，整个时间复杂度O(n)
   */
  private void preOrderTraverseNoRecursive(BinTreeNode root, List result) {
    if (root == null) {
      return;
    }
    BinTreeNode p = root;
    Stack s = new Stack();
    while (p != null) {
      while (p != null) {
        result.add(p);
        if (p.right != null) {
          s.push(p.right);
        }
        p = p.left;
      }
      if (!s.isEmpty()) {
        p = (BinTreeNode)s.pop();
      }
    }
  }

  /**
   * 树的中序遍历
   * 内层循环中，沿着根结点P一直向左走，沿途将根结点入栈，直到p为空，此时应当取出上一层的根结点访问之，
   * 然后转向该根结点的右子树进行中序遍历，如果栈跟p都为空，则说明没有更多的子树需要遍历了，时间复杂度O(n)
   */
  private void inOrderTraverseNoRecursive(BinTreeNode root, List result) {
    if (root == null) {
      return;
    }
    BinTreeNode p = root;
    Stack stack =  new Stack();
    while(p != null || !stack.isEmpty()) {
      while(p != null) {
        stack.push(p);
        p = p.left;
      }
      if (!stack.isEmpty()) {
        p = (BinTreeNode)stack.pop();
        result.add(p);
        p = p.right;
      }
    }
  }

  /**
   * 树的后序遍历
   * 内层的第一个while循环，沿着根结点p向左子树深入，如果左子树为空，则向右子树深入，沿途将根结点入栈，直到p为空
   * 第一个if语句说明应当取出栈顶根结点进行访问，此时栈顶结点为叶子或无右子树的单分支结点，访问p以后，说明p为根的子树访问完毕
   * 判断p是否为其父结点的右孩子，如果是，则说明只要访问其父亲就可以完成对以p的父亲为根的子树的遍历，以内层的第二个while循环完成
   * 如果不是，则转向其父结点的右子树继续后序遍历，如果栈和p都为空，则说明结束了。时间复杂度为O(n)
   */
  private void postOrderTraverseNoRecursive(BinTreeNode root, List result) {
    if (root == null) {
      return;
    }
    BinTreeNode p = root;
    Stack stack =  new Stack();
    while (p != null || !stack.isEmpty()) {
      while (p != null) { //先左后右不断深入
        stack.push(p);    //将根结点入栈
        if (p.left != null) {
          p = p.left;
        } else {
          p = p.right;
        }
      }

      if (!stack.isEmpty()) {
        p = (BinTreeNode)stack.pop(); //取出栈顶根结点访问之
        result.add(p);
      }

      //满足条件时，说明栈顶根结点右子树已访问，应出栈访问之
      while (!stack.isEmpty() && ((BinTreeNode)stack.peek()).right == p) {
        p = (BinTreeNode)stack.pop();
        result.add(p);
      }
      //转向栈顶根结点的右子树继续后序遍历
      if (!stack.isEmpty()) {
        p = ((BinTreeNode)stack.peek()).right;
      } else {
        p = null;
      }
    }
  }

  /**
   * 树的层次遍历
   * 用一个队列实现
   */
  private void levelOrderTraverseNoRecursive(BinTreeNode root, List result) {
    if (root == null) {
      return;
    }
    Queue queue = new LinkedList();
    queue.offer(root);
    while(!queue.isEmpty()) {
      BinTreeNode b = (BinTreeNode)queue.poll();
      result.add(b);
      if (b.left != null) {
        queue.offer(b.left);
      }
      if (b.right != null) {
        queue.offer(b.right);
      }
    }
  }

}

class BinTreeNode {

  int val;
  BinTreeNode left;
  BinTreeNode right;

  public BinTreeNode(int val) {
    this.val = val;
    this.left = null;
    this.right = null;
  }
}

```

#### 20.二叉树展开成链表（LC）

题目：将二叉树展开成链表，全是右子树形式

思路：找到规律，将当前节点的右子树放到左子树的最右边的节点上，同时左子树要清空

```java
package com.qiming.algorithm.leetcode;

/**
 * 给定一个二叉树，将其展开成链表（都是右子树）
 * 找规律。将当前节点的右子树放到左子树的最右边的节点上
 */
public class BinTreeToList {

  public void flatten(TreeNode root) {
    for (TreeNode p = root; p != null; p=p.right) {
      if (p.left == null) {
        continue;
      }
      TreeNode most_right = p.left;
      while (most_right.right != null) {
        most_right = most_right.right;
      }
      most_right.right = p.right;
      p.right = p.left;
      p.left = null;
    }
  }

}

```

#### 21.万万没想到之抓捕孔连顺（NC）

题目：计算所有可能的方案

思路：最左边的值固定，再从剩下的元素中Cn2

```java
package com.qiming.algorithm.nowcoder;

import java.util.LinkedList;
import java.util.List;
import java.util.Scanner;

/**
 *  万万没想到之抓捕孔连顺
 *  第一行包含空格分隔的两个数字 N和D(1 ≤ N ≤ 1000000; 1 ≤ D ≤ 1000000)
 *  第二行包含N个建筑物的的位置，每个位置用一个整数（取值区间为[0, 1000000]）表示，从小到大排列（将字节跳动大街看做一条数轴）
 *  输出一个数字，表示不同埋伏方案的数量。结果可能溢出，请对 99997867 取模
 */
public class FindScheme {

  public static void main(String[] args) {
//    Scanner scanner = new Scanner(System.in);
//
//    String str1 = scanner.nextLine();
//    String[] array1 = str1.split(" ");
//    int N = Integer.parseInt(array1[0]);
//    int D = Integer.parseInt(array1[1]);
//
//    String str = scanner.nextLine();
//    String[] array = str.split(" ");
//
//    List list = new LinkedList();
//
//    for (int i = 0; i < array.length; i++) {
//      list.add(Integer.parseInt(array[i]));
//    }
//
//    System.out.println(countScheme(list, D));

    /**
     * 可以这样拿int
      */
    Scanner sc = new Scanner(System.in);
    int N = sc.nextInt();
    int D = sc.nextInt();
    int[] dist = new int[N];
    for (int i = 0; i < N; i++) {
      dist[i] = sc.nextInt();
    }
    long i = totalProgram(dist, D);
    System.out.println(i);

  }


  /**
   * 此方法通过，computeCn一定要用long的参数和输出都要用long
   * int的区间[2147483647 -2147483648]，随便输个998899，那么computeCn的值是498,899,106,651‬
   * 最左边的值固定，再从剩下的元素中Cn2
   * @param dist
   * @param D
   * @return
   */
  public static long totalProgram(int[] dist, int D) {
    long ans = 0;
    for (int i = 0,j = 0;i < dist.length;i++){
      while (i >= 2 && (dist[i] - dist[j]) > D)
        j++;
      ans += computeCn(i - j);
    }
    return ans % 99997867;
  }
  private static long computeCn(long n) {
    return n * (n - 1) / 2;
  }


  /**
   * 测试不通过，使用滑动窗口的
   * @param array
   * @param val
   * @return
   */
  public static int countSchemeWindow(int[] array, int val) {
    int count = 0;
    int left = 0;
    int right = 2;
    while (left < array.length - 2) {
      while (right < array.length && array[right] - array[left] <= val) {
        right += 1;
      }
      if (right - 1 - left >= 2) {
        int num = right - left - 1;
        count = count + num * (num - 1);
      }
      left += 1;
    }
    return count % 99997867;
  }

  /**
   * 测试超时，不管是list还是数组存储的
   * 主要原因是要看清题，是从小到大排列的
   * @param list
   * @param val
   * @return
   */
  public static int countScheme(List list, int val) {
    System.out.println(System.currentTimeMillis());
    int count = 0;
    int[] s = new int[3];
    for (int i = 0; i < list.size(); i++) {
      s[0] = (Integer)list.get(i);
      for (int j = i + 1; j < list.size(); j++) {
        s[1] = (Integer)list.get(j);
        if (Math.abs(s[0] - s[1]) > val) {
          continue;
        }
        for (int k = j + 1; k < list.size(); k++) {
          s[2] = (Integer)list.get(k);
          if (Math.abs(s[0] - s[2]) > val || Math.abs(s[1] - s[2]) > val) {
            continue;
          } else {
            count++;
          }
        }
      }
    }
    System.out.println(System.currentTimeMillis());
    return Math.floorMod(count, 99997867);
  }

}

```

#### 22.特征提取（NC）

题目：小明是一名算法工程师，同时也是一名铲屎官。某天，他突发奇想，想从猫咪的视频里挖掘一些猫咪的运动信息。为了提取运动信息，他需要从视频的每一帧提取“猫咪特征”。一个猫咪特征是一个两维的vector<x, y>。如果x_1=x_2 and y_1=y_2，那么这俩是同一个特征。因此，如果喵咪特征连续一致，可以认为喵咪在运动。也就是说，如果特征<a, b>在持续帧里出现，那么它将构成特征运动。比如，特征<a, b>在第2/3/4/7/8帧出现，那么该特征将形成两个特征运动2-3-4 和7-8。现在，给定每一帧的特征，特征的数量可能不一样。小明期望能找到最长的特征运动。

思路：暴力破解了

```java
package com.qiming.algorithm.nowcoder;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.Scanner;

/**
 * 特征提取
 * 第一行包含一个正整数N，代表测试用例的个数。
 * 每个测试用例的第一行包含一个正整数M，代表视频的帧数。
 * 接下来的M行，每行代表一帧。其中，第一个数字是该帧的特征个数，接下来的数字是在特征的取值；比如样例输入第三行里，2代表该帧有两个猫咪特征，<1，1>和<2，2>
 * 所有用例的输入特征总数和<100000
 * N满足1≤N≤100000，M满足1≤M≤10000，一帧的特征个数满足 ≤ 10000。
 * 特征取值均为非负整数。
 * 思路是用linkedHashMap和暴力破解
 */
public class FeatureExtract {

  public static void main(String[] args) {

    Scanner scanner = new Scanner(System.in);
    int result = 0;
    int sampleCount = scanner.nextInt();
    for (int i = 0; i < sampleCount; i++) {
      int frameCount = scanner.nextInt();
      LinkedHashMap lhm = new LinkedHashMap(frameCount);
      for (int j = 0; j < frameCount; j++) {
        int vectorCount = scanner.nextInt();
        Vector[] vector = new Vector[vectorCount];
        for (int k = 0; k < vectorCount; k++) {
          int x = scanner.nextInt();
          int y = scanner.nextInt();
          Vector v = new Vector(x, y);
          vector[k] = v;
        }
        lhm.put(j, vector);
      }
      int thisSampleCount = frameList(lhm);
      result = Math.max(result, thisSampleCount);
    }
    System.out.println(result);
  }

  public static int frameList(Map map) {
    int result = 0;
    for (int i = 0; i < map.size(); i++) {
      Vector[] vector = (Vector[])map.get(i);
      for (int j = 0; j < vector.length; j++) {
        Vector toCompare = vector[j];
        int tmpResult = 1;
        int k = i + 1;
        while(k < map.size()) {
          Vector[] nextVector = (Vector[])map.get(k);
          boolean hasNext = false;
          for (int m = 0; m < nextVector.length; m++) {
            if (nextVector[m].x == toCompare.x && nextVector[m].y == toCompare.y) {
              tmpResult = tmpResult + 1;
              k++;
              hasNext = true;
            }
          }
          if (!hasNext) {
            break;
          }
        }
        result = Math.max(result, tmpResult);
      }
    }
    return result;
  }


}

class Vector{
  int x;
  int y;

  public Vector(int x, int y) {
    this.x = x;
    this.y = y;
  }
}
```

#### 23.明明的随机数（NC）

题目：明明想在学校中请一些同学一起做一项问卷调查，为了实验的客观性，他先用计算机生成了N个1到1000之间的随机整数（N≤1000），对于其中重复的数字，只保留一个，把其余相同的数去掉，不同的数对应着不同的学生的学号。然后再把这些数从小到大排序，按照排好的顺序去找同学做调查。请你协助明明完成“去重”与“排序”的工作(同一个测试用例里可能会有多组数据，希望大家能正确处理)。

思路：直接用现成集合算了

```java
package com.qiming.HW;

import java.util.Set;
import java.util.TreeSet;
import java.util.Scanner;
import java.util.Iterator;

/**
 * 明明想在学校中请一些同学一起做一项问卷调查，为了实验的客观性，他先用计算机生成了N个1到1000之间的随机整数（N≤1000），
 * 对于其中重复的数字，只保留一个，把其余相同的数去掉，不同的数对应着不同的学生的学号。然后再把这些数从小到大排序，
 * 按照排好的顺序去找同学做调查。请你协助明明完成“去重”与“排序”的工作(同一个测试用例里可能会有多组数据，希望大家能正确处理)。
 *
 * 这个思路，，直接用个集合吧，，都不用写函数，，注意while循环去接收一个测试用例里的多组数据，这样提交代码才能过
 */
public class RemoveAndSort {

  public static void main(String args[]) {
    Scanner scanner = new Scanner(System.in);
    while (scanner.hasNextInt()) {
      int num = scanner.nextInt();
      Set set = new TreeSet();
      for (int i = 0; i < num; i++) {
        set.add(scanner.nextInt());
      }
      Iterator iterator = set.iterator();
      while (iterator.hasNext()) {
        System.out.println(iterator.next());
      }
    }
  }

}

```

#### 24.字符串分割（NC）

题目：•连续输入字符串，请按长度为8拆分每个字符串后输出到新的字符串数组；•长度不是8整数倍的字符串请在后面补数字0，空字符串不处理，连续输入字符串(输入2次,每个字符串长度小于100)，输出到长度为8的新字符串数组

思路：直接用String进行操作

```java
package com.qiming.HW;

import java.util.Scanner;

/**
 * 连续输入字符串，请按长度为8拆分每个字符串后输出到新的字符串数组
 * 长度不是8整数倍的字符串请在后面补数字0，空字符串不处理。
 *
 * 连续输入字符串(输入2次,每个字符串长度小于100)
 *
 * 思路就是String的使用，注意这里面不需要一个测试用例有多组数据
 */
public class HandleTwoString {

  public static void main(String args[]) {
    Scanner scanner = new Scanner(System.in);
    String s1 = scanner.nextLine();
    String s2 = scanner.nextLine();
    String[] str1 = handle(s1);
    String[] str2 = handle(s2);
    if (str1 != null) {
      for (int i = 0; i < str1.length; i++) {
        System.out.println(str1[i]);
      }
    }
    if (str2 != null) {
      for (int i = 0; i < str2.length; i++) {
        System.out.println(str2[i]);
      }
    }
  }

  /**
   * 这个方法不好
   * @param str
   * @return
   */
  private static String[] handle(String str) {
    if (str == null || str.equals("")) {
      return null;
    }

    char[] chars = str.toCharArray();
    if (chars.length % 8 == 0) {
      int arraynum = chars.length / 8;
      String[] s = new String[arraynum];
      for (int i = 0; i < arraynum; i++) {
        char[] temp = new char[8];
        for (int j = 0 ; j < temp.length; j++) {
          temp[j] = chars[i*8+j];
        }
        s[i] = String.valueOf(temp);
      }
      return s;
    } else {
      int arraynum = chars.length / 8 + 1;
      int havenum = chars.length % 8;
      String[] s = new String[arraynum];
      for (int i = 0; i < arraynum - 1; i++) {
        char[] temp = new char[8];
        for (int j = 0 ; j < temp.length; j++) {
          temp[j] = chars[i*8+j];
        }
        s[i] = String.valueOf(temp);
      }
      char[] last = new char[8];;
      for (int i = 0; i < havenum; i++) {
        last[i] = chars[(arraynum-1) * 8 + i];
      }
      for (int j = 7; j > havenum - 1; j--) {
        last[j] = '0';
      }
      //再补充最后的
      s[arraynum-1] = String.valueOf(last);
      return s;
    }
  }

  /**
   * 直接用字符串处理比较清晰
   * @param str
   * @return
   */
  private static String[] handleByString(String str) {
    if (str == null || str.equals("")) {
      return null;
    }

    int length = str.length();
    if (length % 8 == 0) {
      int arraynum = length / 8;
      String[] s = new String[arraynum];
      for (int i = 0; i < arraynum; i++) {
        s[i] = str.substring(i*8, i*8 + 8);
      }
      return s;
    } else {
      int arraynum = length / 8 + 1;
      int havanum = length % 8;
      String[] s = new String[arraynum];
      for (int i = 0; i < arraynum - 1; i++) {
        s[i] = str.substring(i*8, i*8 + 8);
      }
      //补充最后的
      s[arraynum-1] = str.substring(8*(arraynum-1), length);
      for (int i = 0; i < 8- havanum; i++) {
        s[arraynum-1] = s[arraynum-1] + "0";
      }
      return s;
    }
  }

}

```

#### 25.计算字符串中没有重复字符的字串的最长长度

题目：计算字符串中没有重复字符的字串的最长长度

思路：很多问题中提到的滑动窗口

```java
package com.qiming.algorithm.al;

import java.util.HashSet;
import java.util.Scanner;

/**
 * longest substring without repeating characters.
 * Given a string, find the length of the longest substring without repeating characters.
 */
public class LongestStringLength {

  public static void main(String[] args) {
    //输入测试
    Scanner in = new Scanner(System.in);
    while (in.hasNextLine()) {
      String input = in.nextLine();
      System.out.println(LengthOfLongestSubstring(input));
    }
  }

  public static int LengthOfLongestSubstring(String str) {
    //空字串返回0
    int result = 0;
    int strLength = str.length();
    int i = 0, j = 0;

    HashSet<String> hashSet = new HashSet<String>();

    while (i < strLength && j < strLength)
    {
      String oneStr = str.substring(j, j + 1);
      if (!hashSet.contains(oneStr))
      {
        hashSet.add(oneStr);
        j++;
        result = Math.max(result, j-i);
      }
      else
      {
        String oneStrI = str.substring(i, i + 1);
        hashSet.remove(oneStrI);
        i++;
      }
    }
    return result;
  }

}

```

#### 26.毕业旅行问题（NC）

题目：小明目前在做一份毕业旅行的规划。打算从北京出发，分别去若干个城市，然后再回到北京，每个城市之间均乘坐高铁，且每个城市只去一次。由于经费有限，希望能够通过合理的路线安排尽可能的省一些路上的花销。给定一组城市和每对城市之间的火车票的价钱，找到每个城市只访问一次并返回起点的最小车费花销。

思路：动态规划解题

城市的邻接表如下：
          S0  S1  S2  S3  
   S0   0   3    6     7
   S1   5   0    2     7   
   S2   6   6   0     2
   S3   3   3    5     0
假设找出的一条最短的回路：S0 -> S1 -> S2 -> S3 -> S0
我们可以利用结论： "S1 S2 S3 S0 "必然是从S1到S0通过其它各点的一条最短路径(如果不是，则会出现矛盾)
Length(总回路) = Length(S0，S1)  + Length(S1,S2,S3,S0)

**注意这里的length，最后是要回到s0的，也就是与下面的d函数要保持一致，不然d的理解会容易出现偏差**

从上面的公式把总回路长度分解：
Length(回路) =Min{ Length(0,1)+Length(1,…,0)， Length(0,2)+Length(2,…,0)，Length(0,3)+Length(3,…,0)}
规范化地表达上面的公式
d(i，V) 表示从i点经过点集Ｖ各点一次之后回到出发点的最短距离
d(i，V') ＝ min {Cik+d(k,V－{k})}   (k∈V')
d(k，{ }) ＝ Cik (k≠i)                          
其中，Ｃik表示i到k的距离     
从城市0出发，经城市1、2、3然后回到城市0的最短路径长度是：
d(0, {1, 2, 3})=min{C01+ d(1, { 2, 3}), C02+ d(2, {1, 3}),C03+ d(3, {1, 2})}

这是最后一个阶段的决策，它必须依据d(1, { 2, 3})、
d(2, {1, 3})和d(3, {1, 2})的计算结果,而：

d(1, {2, 3})=min{C12+d(2, {3}), 　C13+ d(3, {2})}
d(2, {1, 3})=min{C21+d(1, {3}), 　C23+ d(3, {1})}
d(3, {1, 2})=min{C31+d(1, {2}), 　C32+ d(2, {1})}
继续写下去：
d(1, {2})= C12+d(2, {})   d(2, {3})=C23+d(3, {})   d(3, {2})= C32+d(2, {})  
d(1, {3})= C13+d(3, {})   d(2, {1})=C21+d(1, {})   d(3, {1})= C31+d(1, {})

建立dp表

![dp]({{ "/assets/img/algorithm/dp_table.png" | relative_url}})

```java
package com.qiming.algorithm.nowcoder;

import java.util.Scanner;

/**
 * 动态规划的问题，旅行家
 */

public class TSP {

  public static void main(String[] args) {

    Scanner in = new Scanner(System.in);
    int cityNum = in.nextInt();// 城市数目
    int[][] dist = new int[cityNum][cityNum];// 距离矩阵，距离为欧式空间距离
    for (int i = 0; i < dist.length; i++)
      for (int j = 0; j < cityNum; j++) {
        dist[i][j] = in.nextInt();
      }
    in.close();

    int V = 1 << (cityNum - 1);// 对1进行左移n-1位，值刚好等于2^(n-1)
    // dp表，n行，2^(n-1)列
    int[][] dp = new int[cityNum][V];
    // 初始化dp表第一列
    for (int i = 0; i < cityNum; i++)  dp[i][0] = dist[i][0];

    //设想一个数组城市子集V[j]，长度为V,且V[j] = j,对于V[j]即为压缩状态的城市集合
    //从1到V-1  用二进制表示的话，刚好可以映射成除了0号城市外的剩余n-1个城市在不在子集V[j]，1代表在，0代表不在
    //若有总共有4个城市的话，除了第0号城市，对于1-3号城市
    //111 = V-1 = 2^3 - 1  = 7 ，从高位到低位表示3到1号城市都在子集中
    //而101 = 5 ，表示3,1号城市在子集中，而其他城市不在子集中
    //这里j不仅是dp表的列坐标值，如上描述，j的二进制表示城市相应城市是否在子集中
    for (int j = 1; j < V; j++)
      for (int i = 0; i < cityNum; i++) { //这个i不仅代表城市号，还代表第i次迭代
        dp[i][j] = Integer.MAX_VALUE; //为了方便求最小值,先将其设为最大值
        if (((j >> (i - 1)) & 1) == 0) {
          // 因为j就代表城市子集V[j],((j >> (i - 1))是把第i号城市取出来
          //并位与上1，等于0，说明是从i号城市出发，经过城市子集V[j]，回到起点0号城市
          for (int k = 1; k < cityNum; k++) { // 这里要求经过子集V[j]里的城市回到0号城市的最小距离
            if (((j >> (k - 1)) & 1) == 1) { //遍历城市子集V[j]
              //设s=j ^ (1 << (k - 1))
              //dp[k][j ^ (1 << (k - 1))，是将dp定位到，从k城市出发，经过城市子集V[s]，回到0号城市所花费的最小距离
              //怎么定位到城市子集V[s]呢，因为如果从k城市出发的，经过城市子集V[s]的话
              //那么V[s]中肯定不包含k了，那么在j中把第k个城市置0就可以了，而j ^ (1 << (k - 1))的功能就是这个
              dp[i][j] = Math.min(dp[i][j], dist[i][k] + dp[k][j ^ (1 << (k - 1))]); //^异或
              //还有怎么保证dp[k][j ^ (1 << (k - 1))]的值已经得到了呢，
              //注意所有的计算都是以dp表为准，从左往右从上往下的计算的，每次计算都用到左边列的数据
              //而dp表是有初试值的，所以肯定能表格都能计算出来
            }
          }
        }
      }
    System.out.println(dp[0][V - 1]);
  }

}

```

#### 27.找零（NC）

题目：Z国的货币系统包含面值1元、4元、16元、64元共计4种硬币，以及面值1024元的纸币。现在小Y使用1024元的纸币购买了一件价值为0<N<=1024的商品，请问最少他会收到多少硬币？

思路：动态规划解题，状态转移方程，DP表，[详见](https://zqmalyssa.github.io/strap/2015/08/01/datastructure-and-algorithm-post.html)

```java
package com.qiming.algorithm.nowcoder;

import java.util.Scanner;

/**
 * Z国的货币系统包含面值1元、4元、16元、64元共计4种硬币，以及面值1024元的纸币。现在小Y使用1024元的纸币购买了一件价值为的商品，请问最少他会收到多少硬币？
 *
 * 思路，这是动态规划题目中经典的找零问题，动态规划之前说了旅行家问题，现在的找零问题需要总结3个方面
 *
 * 暴力解法(递归)，带备忘录的递归算法，动态规划
 */
public class Change {

  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    int cost = scanner.nextInt();
    int nums[] = {1, 4, 16, 64};
    System.out.println(calNumWithRecord(1024 - cost, nums));
  }

  /**
   * 暴力解法，递归，时间复杂度爆了，运行是各种不通过，O(k*n^k)的时间复杂度
   * @param change
   * @param nums
   * @return
   */
  private static int calNum(int change, int nums[]) {
    if (change == 0) {
      return 0;
    }
    int result = Integer.MAX_VALUE;
    for (int num : nums) {
      if (change - num < 0) {
        continue;
      }
      int subPro = calNum(change - num, nums);
      //子问题无解
      if (subPro == -1) {
        continue;
      }
      result = Math.min(result, subPro + 1);
    }
    return result == Integer.MAX_VALUE ? -1 : result;
  }

  /**
   * 带备忘录的递归算法，时间复杂度不爆了，可以运行通过，O(kn)的时间复杂度
   * @param change
   * @param nums
   * @return
   */
  private static int calNumWithRecord(int change, int nums[]) {
    int mem[] = new int[change + 1];
    //所有初始化为-2
    for (int i = 0; i < mem.length; i++) {
      mem[i] = -2;
    }
    return helper(change, nums, mem);
  }

  private static int helper(int change, int nums[], int mem[]) {
    if (change == 0) {
      return 0;
    }
    if (mem[change] != -2) {
      return mem[change];
    }
    int result = Integer.MAX_VALUE;
    for (int num : nums) {
      //金额不可达
      if (change - num < 0) {
        continue;
      }
      int subPro = helper(change - num, nums, mem);
      if (subPro == -1) {
        continue;
      }
      result = Math.min(result, subPro + 1);
    }
    //记录本轮答案
    mem[change] = (result == Integer.MAX_VALUE) ? -1 : result;
    return mem[change];
  }

  /**
   * 动态规划的解法，用到了DP表，这边就是一个数组
   * 非常重要的就是写好状态转移方程
   * @param change
   * @param nums
   * @return
   */
  private static int calNumWithDP(int change, int nums[]) {
    int dp[] = new int[change + 1];
    dp[0] = 0;
    for (int i = 1; i < dp.length; i++) {
      //初始化为change + 1，所以返回结果比较的是change + 1，change + 1应该就是所有组合中的最大值了，不要初始化为Integer.MAX_VALUE
      dp[i] = change + 1;
    }
    for (int i = 0; i < dp.length; i++) {
      //内层循环求所有子问题+1的最小值
      for (int num : nums) {
        if (i - num < 0) {
          continue;
        }
        dp[i] = Math.min(dp[i], 1 + dp[i - num]);
      }
    }
    return (dp[change] == change + 1) ? -1 : dp[change];
  }

}

```

#### 28.N皇后问题

题目：在国际象棋中，皇后是最强大的一枚棋子，可以吃掉与其在同一行、列和斜线的敌方棋子。八皇后问题是这样一个问题：将八个皇后摆在一张8*8的国际象棋棋盘上，使每个皇后都无法吃掉别的皇后，一共有多少种摆法？

思路：思路是采用回溯法

```java
package com.qiming.algorithm;

import java.util.Scanner;

/**
 * N皇后问题，在4*4或者8*8的棋盘上摆放皇后，使得它们之间不会被互相吃掉
 *
 * 思路是采用回溯法，回溯算法将解空间看作一定的结构，通常为树形结构，一个解对应于树中的一片树叶。算法从树根（即初始状态出发），尝试所有可能到达的结点。
 * 当不能前行时就后退一步或若干步，再从另一个结点开始继续搜索，直到尝试完所有的结点。也可以用走迷宫的方式去理解回溯，设想把你放在一个迷宫里，想要走出迷宫，
 * 最直接的办法是什么呢？没错，试。先选一条路走起，走不通就往回退尝试别的路，走不通继续往回退，直到走遍所有的路，并且在走的过程中你可以记录所有能走出迷宫的路线
 */
public class NQueen {

  private static int totalNum = 0;

  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    int n = scanner.nextInt();
    int c[] = new int[n];
    queen(0, c, n);
    System.out.println(totalNum);
  }

  /**
   * 这个函数是用来摆放第row行中的皇后，用c[i]记录第i行的皇后放置的位置j列
   * @param row
   * @param c
   * @param n
   */
  private static void queen(int row, int c[], int n) {
    if (row == n) {
      //当row等于n的时候说明皇后已经全部摆放完毕
      for (int i = 0; i < n; i++) {
        System.out.print(c[i] + " ");
      }
      System.out.println("\n");
      totalNum++;
    } else {
      //皇后还没有摆完
      //遍历所有的列，看row行皇后可以放在第几列
      for (int col = 0; col < n; col++) {
        //要更新数组
        c[row] = col;
        //如果可以放在row行col列则继续摆放下一行
        if (isOk(c, row)) {
          queen(row + 1, c, n);
        }
        //如果循环了所有的列都不能摆放，则会回溯到前一层函数改变上一行皇后的摆放
      }
    }
  }

  /**
   * 判断皇后能不能摆放到这个位置
   * 也就是row行皇后不能和任意之前的皇后在列上 / 和 \方向上一致
   * @param c
   * @param row
   * @return
   */
  private static boolean isOk(int c[], int row) {
    for (int i = 0; i < row; i++) {
      //第row行皇后不能和任意之前的皇后在同一列或 \方向或 / 方向
      if (c[i] == c[row] || c[row]-row == c[i]-i || c[row]+row == c[i]+i) {
        return false;
      }
    }
    return true;
  }


}

```

#### 29.组合总数（LC）

题目：给定一个无重复元素的数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。说明，所有数字（包括 target）都是正整数。解集不能包含重复的组合。

思路：思路还是采用回溯法，回溯法有模板可套，这边有一点小变化，需要有个标志位去从头开始遍历

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.List;

/**
 * 组合总数
 *
 * 给定一个无重复元素的数组 candidates 和一个目标数 target ，找出 candidates 中所有可以使数字和为 target 的组合。说明，所有数字（包括 target）都是正整数。解集不能包含重复的组合。
 *
 * 思路是用回溯法，回溯法有模板可套，这边有一点小变化，需要有个标志位去从头开始遍历
 */
public class CombinedTotal {

  //自己可以这样写，如果是LC上，需要把这个变量变成局部变量
  private static List<List<Integer>> result = new LinkedList<>();

  public static void main(String[] args) {
    LinkedList<Integer> track =  new LinkedList<>();
    int nums[] = {2,3,5,6};
    backtrack(nums, track, 11, 0);

//    int nums[] = {2,3,6,7};
//    backtrack(nums, track, 7, 0);
    for (List<Integer> integers : result) {
      for (Integer integer : integers) {
        System.out.print(integer + " ");
      }
      System.out.println("\n");
    }
  }

  private static void backtrack(int nums[], LinkedList<Integer> track, int target, int begin) {
    int res = 0;
    if (track.size() != 0) {
      //可以lambda表达式写
      res = track.stream().reduce(Integer::sum).get();
    }
//    for (int i = 0; i < track.size(); i++) {
//      res += track.get(i);
//    }

    if (res == target) {
      //如果和等于target了，结束
      result.add(new LinkedList<>(track));
      return;
    }

    for (int i = begin; i < nums.length; i++) {
      //加入之前要排除不合法的选择，这边一定要想清楚，不然会死循环
      if (res + nums[i] > target) {
        continue;
      }
      track.add(nums[i]);
      //这个i是关键，去除重复的解
      backtrack(nums, track, target, i);
      track.removeLast();
    }

  }

}

```
#### 30.组合总数（LC）

题目：给你一棵二叉树，请你返回层数最深的叶子节点的和。

思路：使用层序遍历，计算每一层的和，只要还有下一层累加和就重置，最后结果就是最后一层的和

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 层数最深叶子节点的和
 *
 * 思路 使用层序遍历，计算每一层的和，只要还有下一层累加和就重置，最后结果就是最后一层的和
 */
public class SumOfDeepestLeave {

  public static void main(String[] args) {
    //自己写测试用例吧
  }

  private static int sumOfeepestLeave(TreeNodeSumOfDeepestLeave root) {
    if (root == null) {
      return 0;
    }
    Queue queue = new LinkedList();
    queue.offer(root);

    //改造基本的树的层次遍历
    int result = 0;
    while (!queue.isEmpty()) {
      int size = queue.size();
      result = 0; //每一层先置为0
      for (int i = 0; i < size; i++) {
        TreeNodeSumOfDeepestLeave p = (TreeNodeSumOfDeepestLeave)queue.poll();
        result += p.val;
        if (p.left != null) {
          queue.offer(p.left);
        }
        if (p.right != null) {
          queue.offer(p.right);
        }
      }
    }
    return result;
  }

}


class TreeNodeSumOfDeepestLeave {

  int val;
  TreeNodeSumOfDeepestLeave left;
  TreeNodeSumOfDeepestLeave right;
  public TreeNodeSumOfDeepestLeave(int val) {
    this.val = val;
    left = null;
    right = null;
  }

}
```

#### 31.二叉树的锯齿形层次遍历（LC）

题目：给定一个二叉树，返回其节点值的锯齿形层次遍历。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。

思路：将层次遍历的队列换成栈，然后需要交叉访问左右子树的顺序

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;
import java.util.Stack;

/**
 * 二叉树的锯齿形层次遍历
 *
 * 给定一个二叉树，返回其节点值的锯齿形层次遍历。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。
 *
 * 思路是将层次遍历的队列换成栈，然后需要交叉访问左右子树的顺序
 */
public class BinaryTreeZigzag {

  public static void main(String[] args) {
    TreeNodeBinaryTreeZigzag root = new TreeNodeBinaryTreeZigzag(1);
    TreeNodeBinaryTreeZigzag left1 = new TreeNodeBinaryTreeZigzag(2);
    TreeNodeBinaryTreeZigzag right1 = new TreeNodeBinaryTreeZigzag(3);
    TreeNodeBinaryTreeZigzag left2 = new TreeNodeBinaryTreeZigzag(4);
    TreeNodeBinaryTreeZigzag right2 = new TreeNodeBinaryTreeZigzag(5);
    root.left = left1;
    root.right = right1;
    left1.left = left2;
    left1.right = right2;

    new BinaryTreeZigzag().zigzagLevelOrder(root);
  }

  public List<List<Integer>> zigzagLevelOrder(TreeNodeBinaryTreeZigzag root) {
    if (root == null) {
      return new LinkedList<>();
    }

    List<List<Integer>> result = new LinkedList<>();

    Stack stack = new Stack();
    stack.push(root);
    boolean rightThenLeft = false;
    while (!stack.isEmpty()) {
      //每一层的size
      int size = stack.size();
      //存储结果
      List<Integer> list = new LinkedList<>();
      //存储层的结点
      Queue queue = new LinkedList<>();
      for (int i = 0; i < size; i++) {
        TreeNodeBinaryTreeZigzag p = (TreeNodeBinaryTreeZigzag)stack.pop();
        list.add(p.val);
        queue.offer(p);
      }
      //加入集合
      result.add(list);
      //入栈

      while (!queue.isEmpty()) {
        TreeNodeBinaryTreeZigzag q = (TreeNodeBinaryTreeZigzag)queue.poll();
        //入栈之前需要交叉变化
        if (rightThenLeft) {
          if (q.right != null) {
            stack.push(q.right);
          }

          if (q.left != null) {
            stack.push(q.left);
          }

        } else {
          if (q.left != null) {
            stack.push(q.left);
          }

          if (q.right != null) {
            stack.push(q.right);
          }

        }

      }
      //交叉左右顺序
      rightThenLeft = !rightThenLeft;

    }
    return result;
  }

}


class TreeNodeBinaryTreeZigzag {

  int val;
  TreeNodeBinaryTreeZigzag left;
  TreeNodeBinaryTreeZigzag right;
  public TreeNodeBinaryTreeZigzag(int val) {
    this.val = val;
  }

}
```

#### 32.爱吃香蕉的珂珂（LC）

题目：珂珂喜欢吃香蕉。这里有 N 堆香蕉，第 i 堆中有 piles[i] 根香蕉。警卫已经离开了，将在 H 小时后回来。珂珂可以决定她吃香蕉的速度 K （单位：根/小时）。每个小时，她将会选择一堆香蕉，从中吃掉 K 根。如果这堆香蕉少于 K 根，她将吃掉这堆的所有香蕉，然后这一小时内不会再吃更多的香蕉。 珂珂喜欢慢慢吃，但仍然想在警卫回来前吃掉所有的香蕉。返回她可以在 H 小时内吃掉所有香蕉的最小速度 K（K 为整数）

思路：二分查找遍历结果集

```java
package com.qiming.algorithm.leetcode;

/**
 * 爱吃香蕉的珂珂
 *
 * 珂珂喜欢吃香蕉。这里有 N 堆香蕉，第 i 堆中有 piles[i] 根香蕉。警卫已经离开了，将在 H 小时后回来。珂珂可以决定她吃香蕉的速度 K （单位：根/小时）。
 * 每个小时，她将会选择一堆香蕉，从中吃掉 K 根。如果这堆香蕉少于 K 根，她将吃掉这堆的所有香蕉，然后这一小时内不会再吃更多的香蕉。 珂珂喜欢慢慢吃，但仍然想在警卫回来前吃掉所有的香蕉。
 * 返回她可以在 H 小时内吃掉所有香蕉的最小速度 K（K 为整数）
 *
 * 输入: piles = [3,6,7,11], H = 8  输出: 4
 *
 * 1 <= piles.length <= 10^4，piles.length <= H <= 10^9，1 <= piles[i] <= 10^9 = 1_000_000_000
 *
 * 思路，二分查找解决问题
 */
public class KokoEatingBananas {

  public static void main(String[] args) {
    System.out.println(1_000_000_000);
    System.out.println(Integer.MAX_VALUE);
  }

  /**
   * 如果珂珂能以 K 的进食速度最终吃完所有的香蕉（在 H 小时内），那么她也可以用更快的速度吃完。当珂珂能以 K 的进食速度吃完香蕉时，
   * 我们令 possible(K) 为 true，那么就存在 X 使得当 K >= X 时， possible(K) = True。
   * 举个例子，当初始条件为 piles = [3, 6, 7, 11] 和 H = 8 时，存在 X = 4 使得 possible(1) = possible(2) = possible(3) = False，
   * 且 possible(4) = possible(5) = ... = True。
   * 我们可以二分查找 possible(K) 的值来找到第一个使得 possible(X) 为 True 的 X：这将是我们的答案。我们的循环中，不变量 possible(hi) 总为 True， lo 总小于等于答案
   * @param piles
   * @param H
   * @return
   */
  public int minEatingSpeed(int[] piles, int H) {
    //相当于就是求这个两个值区间的二分查找
    int lo = 1;
    int hi = 1_000_000_000; //最大是10^9次方
    //注意这边是小于
    while (lo < hi) {
      int mi = (lo + hi) / 2;
      if (!possible(piles, H, mi))
        lo = mi + 1;
      else
        hi = mi;
    }

    return lo;
  }

  // Can Koko eat all bananas in H hours with eating speed K?
  public boolean possible(int[] piles, int H, int K) {
    int time = 0;
    for (int p: piles)
      //一定要有减1和加1
      time += (p-1) / K + 1;
    return time <= H;
  }


}
```

#### 33.删除被覆盖区间（LC）

题目：给你一个区间列表，请你删除列表中被其他区间所覆盖的区间。只有当 c <= a 且 b <= d 时，我们才认为区间 [a,b) 被区间 [c,d) 覆盖。在完成所有删除操作后，请你返回列表中剩余区间的数目。

思路：思路是排序 + 遍历

```java
package com.qiming.algorithm.leetcode;

import java.util.Arrays;
import java.util.Comparator;

/**
 * 删除被覆盖的区间
 *
 * 给你一个区间列表，请你删除列表中被其他区间所覆盖的区间。只有当 c <= a 且 b <= d 时，我们才认为区间 [a,b) 被区间 [c,d) 覆盖。在完成所有删除操作后，请你返回列表中剩余区间的数目。
 *
 * 思路是排序 + 遍历
 * 如果我们将所有区间按照左端点递增排序，那么对于排完序的列表中第i个区间(li, ri);
 * 在i之前的区间j，一定满足lj < li，因此只要存在rj > ri，那么i这个区间一定会被覆盖，也就是之前区间的右端点最大值rmax满足>=ri，i就能被覆盖
 * 那么之后的区间呢，对于k，只有lk > li，因此一定不会被后面的区间覆盖，但是如果lk = li呢，还是有可能的，所以将右端点递减，这样即使出现相同的情况，也不会有问题
 */
public class RemoveCoveredIntervals {

  public static void main(String[] args) {
    int a[] = {3,1,2,6};
    int b[][] ={5,2},{4,3},{2,5},{2,4},{4,6} //这边jekyll的问题，这样吧
//    Arrays.sort(b);
//    for (int i = 0; i < b.length; i++) {
//      System.out.println(b[i]);
//    }
    new RemoveCoveredIntervals().removeCoveredIntervals(b);
    for (int i = 0; i <b.length; i++) {
      for (int j = 0; j < b[i].length; j++) {
        System.out.println(b[i][j]);
      }
    }
  }

  public int removeCoveredIntervals(int[][] intervals) {
    //java的二维数组升序降序方法
    Arrays.sort(intervals, new Comparator<int[]>() {
      @Override
      public int compare(int[] o1, int[] o2) {
        return o1[0] == o2[0] ? o2[1] - o1[1] : o1[0] - o2[0];
      }
    });
    int result = intervals.length;
    int rmax = intervals[0][1];
    for (int i = 1; i < intervals.length; i++) {
      if (intervals[i][1] <= rmax) {
        result--;
      } else {
        rmax = Math.max(rmax, intervals[i][1]);
      }
    }
    return result;
  }

  private void sort(int[][] array) {
    quickSortTwoArray(array, 0 , array.length - 1);
  }

  private int partition(int s[][], int low, int high) {
    int pivot = s[low][0];
    int twoValue = s[low][1];
    while (low < high) {
      while (low < high && s[high][0] >= pivot) {
        high --;
      }
      s[low][0] = s[high][0];
      s[low][1] = s[high][1];
      while (low < high && s[low][0] <= pivot) {
        low ++;
      }
      s[high][0] = s[low][0];
      s[high][1] = s[low][1];
    }
    //pivot要被填到坑里面
    s[low][0] = pivot;
    s[low][1] = twoValue;
    return low;
  }

  public void quickSortTwoArray(int s[][], int low, int high) {
    if (low < high) {
      int pa = partition(s, low, high);
      quickSortTwoArray(s, low, pa-1);
      quickSortTwoArray(s, pa+1, high);
    }
  }

}

```

#### 34.掷骰子的N种方法（LC）

题目：这里有 d 个一样的骰子，每个骰子上都有 f 个面，分别标号为 1, 2, ..., f。我们约定：掷骰子的得到总点数为各骰子面朝上的数字的总和。如果需要掷出的总点数为 target，请你计算出有多少种不同的组合情况（所有的组合情况总共有 f^d 种），模 10^9 + 7 后返回。

思路：回溯法不可取，用DP，求状态转移方程，dp[i][j]代表扔i个骰子和为j的所有可能，方程dp[i][j]与dp[i-1]的关系是什么呢，第i次我投了k(1<=k<=f)，那么前i-1次和为j-k，对应dp[i-1][j-k]，于是最终方程是 dp[i][j] = dp[i-1][j-1] + dp[i-1][j-2] + ... + dp[i-1][j-f]，这些都是可能值，边界条件 dp[1][k] = 1 (1<=k<=min(target, f))

```java
package com.qiming.algorithm.leetcode;

import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;

/**
 * 掷骰子的N种方法
 *
 * 这里有 d 个一样的骰子，每个骰子上都有 f 个面，分别标号为 1, 2, ..., f。我们约定：掷骰子的得到总点数为各骰子面朝上的数字的总和。
 * 如果需要掷出的总点数为 target，请你计算出有多少种不同的组合情况（所有的组合情况总共有 f^d 种），模 10^9 + 7 后返回。
 *
 * 思路 不是用回溯，用DP
 * 求状态转移方程，dp[i][j]代表扔i个骰子和为j的所有可能，方程dp[i][j]与dp[i-1]的关系是什么呢，第i次我投了k(1<=k<=f)，那么前i-1次和为j-k，对应dp[i-1][j-k]
 * 于是最终方程是 dp[i][j] = dp[i-1][j-1] + dp[i-1][j-2] + ... + dp[i-1][j-f]，这些都是可能值，边界条件 dp[1][k] = 1 (1<=k<=min(target, f))
 */
public class NumberOfDiceRollsWithTargetSum {

  private int result;

  private static final int MOD = 1000000007;

  public static void main(String[] args) {
//    int result = new NumberOfDiceRollsWithTargetSum().numRollsToTarget(2, 6, 7);
//    int result = new NumberOfDiceRollsWithTargetSum().numRollsToTarget(2, 5, 10);
    int result = new NumberOfDiceRollsWithTargetSum().numRollsToTarget(3, 6, 14);
//    int result = new NumberOfDiceRollsWithTargetSum().numRollsToTarget(30, 30, 500);
    System.out.println(result);
  }

  public int numRollsToTarget(int d, int f, int target) {
    /**
     * 以下是回溯的写法
     */
//    int nums[] = new int[f];
//    for (int i = 0; i < f; i++) {
//      nums[i] = i + 1;
//    }
//    LinkedList<Integer> track = new LinkedList<>();
//    backtrack(nums, track, target, d);
//    return result;

    /**
     * 以下是动态规划的写法
     */
    int [][]dp = new int[31][1001]; //这边是比所有用例多一例
    int min = Math.min(f, target);
    for (int i = 1; i <= min; i++) {
      dp[1][i] = 1;
    }
    //可能的最大值
    int targetMax = d * f;
    for (int i = 2; i <= d ; i++) {
      for (int j = i; j <= targetMax; j++) {
        for (int k = 1; j - k >= 0 && k <= f ; k++) {
          dp[i][j] = (dp[i][j] + dp[i-1][j-k]) % MOD;
        }
      }
    }
    return dp[d][target];
  }

  /**
   * 回溯模板法，在用例为[30,30,500]的情况下，超时了，因为穷举之后的数量是f^d
   * @param nums
   * @param list
   * @param target
   * @param d
   */
  private void backtrack(int nums[], LinkedList<Integer> list, int target, int d) {
    int res = 0;
    if (list.size() == d) {
      for (int i = 0; i < list.size(); i++) {
        res += list.get(i);
      }
      if (target == res) {
        result++;
        return;
      } else {
        return;
      }
    }

    for (int j = 0; j < nums.length; j++) {
      if (res + nums[j] > target) {
        continue;
      }
      list.add(nums[j]);
      backtrack(nums, list, target, d);
      list.removeLast();
    }

  }

}

```

#### 35.四数相加 II（LC）

题目：给定四个包含整数的数组列表 A , B , C , D ,计算有多少个元组 (i, j, k, l) ，使得 A[i] + B[j] + C[k] + D[l] = 0。为了使问题简单化，所有的 A, B, C, D 具有相同的长度 N，且 0 ≤ N ≤ 500 。所有整数的范围在 -2^28 到 2^28 - 1 之间，最终结果不会超过 2^31 - 1

思路：我们以存AB两数组之和为例。首先求出A和B任意两数之和sumAB，以sumAB为key，sumAB出现的次数为value，存入hashmap中。然后计算C和D中任意两数之和的相反数sumCD，在hashmap中查找是否存在key为sumCD。

```java
package com.qiming.algorithm.leetcode;

import java.util.HashMap;
import java.util.Map;

/**
 * 四数相加 II
 *
 * 给定四个包含整数的数组列表 A , B , C , D ,计算有多少个元组 (i, j, k, l) ，使得 A[i] + B[j] + C[k] + D[l] = 0。
 * 为了使问题简单化，所有的 A, B, C, D 具有相同的长度 N，且 0 ≤ N ≤ 500 。所有整数的范围在 -2^28 到 2^28 - 1 之间，最终结果不会超过 2^31 - 1
 *
 * 思路 我们以存AB两数组之和为例。首先求出A和B任意两数之和sumAB，以sumAB为key，sumAB出现的次数为value，存入hashmap中。
 * 然后计算C和D中任意两数之和的相反数sumCD，在hashmap中查找是否存在key为sumCD。
 *
 */
public class FourSumII {

  public static void main(String[] args) {
    int[] A = {1, 2};
    int[] B = {-2, -1};
    int[] C = {-1, 2};
    int[] D = {0, 2};

    int result = new FourSumII().fourSumCount(A, B, C, D);
    System.out.println(result);

  }

  public int fourSumCount(int[] A, int[] B, int[] C, int[] D) {
    Map<Integer, Integer> map = new HashMap<>();
    int res = 0;
    for (int i = 0; i < A.length; i++) {
      for (int j = 0; j < B.length; j++) {
        int sumAB = A[i] + B[j];
        if (map.containsKey(sumAB)) {
          map.put(sumAB, map.get(sumAB) + 1);
        } else {
          map.put(sumAB, 1);
        }
      }
    }

    for (int i = 0; i < C.length; i++) {
      for (int j = 0; j < D.length; j++) {
        int sumCD = -(C[i] + D[j]);
        if (map.containsKey(sumCD)) {
          res += map.get(sumCD);
        }
      }
    }

    return res;
  }


}

```

#### 36.水壶问题（LC）

题目：有两个容量分别为 x升 和 y升 的水壶以及无限多的水。请判断能否通过使用这两个水壶，从而可以得到恰好 z升 的水？如果可以，最后请用以上水壶中的一或两个来盛放取得的 z升 水。

思路：也就是先要求出x和y的最大公约数

```java
package com.qiming.algorithm.leetcode;

/**
 * 水壶问题
 *
 * 有两个容量分别为 x升 和 y升 的水壶以及无限多的水。请判断能否通过使用这两个水壶，从而可以得到恰好 z升 的水？
 * 如果可以，最后请用以上水壶中的一或两个来盛放取得的 z升 水。
 *
 * 你允许：1 装满任意一个水壶 2 清空任意一个水壶 3 从一个水壶向另外一个水壶倒水，直到装满或者倒空
 *
 * 思路 ax + by = z 求是否有合理的解 ，x ，y 为系数，化简 a * t1 * k + b * t2 * k == z;，然后 k * (a * t1 + b * t2) = z;
 * 也就是说z为 a 和 b 的gcd 的倍数 特判为 0 的时候 以及 使得等式成立的基本条件 x + y >= z
 * 也就是先要求出x和y的最大公约数
 */
public class WaterAndJugProblem {

  /**
   * x + y是z的倍数
   * @param x
   * @param y
   * @param z
   * @return
   */
  public boolean canMeasureWater(int x, int y, int z) {
    if (x == 0 && y == 0) {
      return z == 0;
    }
    return z == 0 || (z % gcd(x, y) == 0 && x + y >= z);
  }

  /**
   * 计算x， y的最大公约数
   * @param x
   * @param y
   * @return
   */
  private int gcd(int x, int y) {
    if (y == 0) {
      return x;
    }
    int r = x % y;
    return gcd(y, r);
  }

}

```

#### 37.太平洋大西洋水流问题（LC）

题目：给定一个 m x n 的非负整数矩阵来表示一片大陆上各个单元格的高度。“太平洋”处于大陆的左边界和上边界，而“大西洋”处于大陆的右边界和下边界。规定水流只能按照上、下、左、右四个方向流动，且只能从高到低或者在同等高度上流动。请找出那些水流既可以流动到“太平洋”，又能流动到“大西洋”的陆地单元的坐标。

思路：反向思维么，可以从最左边和最上面开始dfs所有可以到达的山脉，即为可以流向太平洋的山脉，从最右边和最下面开始dfs所有可以到达的山脉，即为可以流向大西洋的山脉。注意：此时dfs的方向为从高度低的山脉向高度高的山脉走

```java
package com.qiming.algorithm.leetcode;

import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;

/**
 * 太平洋大西洋水流问题
 *
 * 给定一个 m x n 的非负整数矩阵来表示一片大陆上各个单元格的高度。“太平洋”处于大陆的左边界和上边界，而“大西洋”处于大陆的右边界和下边界
 * 规定水流只能按照上、下、左、右四个方向流动，且只能从高到低或者在同等高度上流动。请找出那些水流既可以流动到“太平洋”，又能流动到“大西洋”的陆地单元的坐标。
 *
 * 输出坐标的顺序不重要，m 和 n 都小于150
 *
 * 思路 反向思维么，可以从最左边和最上面开始dfs所有可以到达的山脉，即为可以流向太平洋的山脉，从最右边和最下面开始dfs所有可以到达的山脉，即为可以流向大西洋的山脉
 * 注意：此时dfs的方向为从高度低的山脉向高度高的山脉走
 *
 */
public class PacificAtlanticWaterFlow {

  private boolean[][] pacific; // 可以流向太平洋的位置
  private boolean[][] atlantic; // 可以流向大西洋的位置
  private int row, col;

  public static void main(String[] args) {
    int test[][] = (1,2,2,3,5),(3,2,3,4,4),(2,4,5,3,1),(6,7,1,4,5),(5,1,1,2,4); //这边jekyll的问题，这样吧
    List<List<Integer>> result = new PacificAtlanticWaterFlow().pacificAtlantic(test);
    for (List<Integer> list : result) {
      for (Integer integer : list) {
        System.out.print(integer + " ");
      }
      System.out.println("\n");
    }
  }

  public List<List<Integer>> pacificAtlantic(int[][] matrix) {
    List<List<Integer>> res = new LinkedList<>();
    if (matrix == null || matrix.length == 0)
      return new LinkedList<>();
    row = matrix.length;
    col = matrix[0].length;

    pacific = new boolean[row][col];
    atlantic = new boolean[row][col];

    //横边的DFS遍历
    for (int i = 0; i < row; i++) {
      dfs(matrix, pacific, i, 0);
      dfs(matrix, atlantic, i, col - 1);
    }

    //竖边的DFS遍历
    for (int i = 0; i < col; i++) {
      dfs(matrix, pacific, 0, i);
      dfs(matrix, atlantic, row - 1, i);
    }

    //与为1的就是ok的
    for (int i = 0; i < row; i++) {
      for (int j = 0; j < col; j++) {
        if (pacific[i][j] && atlantic[i][j]) {
          LinkedList<Integer> list = new LinkedList();
          list.add(i);
          list.add(j);
          res.add(list);
        }
      }
    }
    return res;
  }

  private void dfs(int[][] matrix, boolean[][] map, int i, int j) {
    if (i < 0 || i >= row || j < 0 || j >= col || map[i][j])
      return;
    map[i][j] = true;
    //上下左右四种情况，从低到高走
    if (i - 1 >= 0 && matrix[i][j] <= matrix[i - 1][j])
      dfs(matrix, map, i - 1, j);
    if (j - 1 >= 0 && matrix[i][j] <= matrix[i][j - 1])
      dfs(matrix, map, i, j - 1);
    if (i + 1 < row && matrix[i][j] <= matrix[i + 1][j])
      dfs(matrix, map, i + 1, j);
    if (j + 1 < col && matrix[i][j] <= matrix[i][j + 1])
      dfs(matrix, map, i, j + 1);
  }

}

```

#### 38.下一个更大元素 II（LC）

题目：给定一个循环数组（最后一个元素的下一个元素是数组的第一个元素），输出每个元素的下一个更大元素。数字 x 的下一个更大的元素是按数组遍历顺序，这个数字之后的第一个比它更大的数，这意味着你应该循环地搜索它的下一个更大的数。如果不存在，则输出 -1。

思路：一个解法是正常思维解法，还有一个解法是单调栈。

```java
package com.qiming.algorithm.leetcode;

import java.util.Stack;

/**
 * 下一个更大元素 II
 *
 * 给定一个循环数组（最后一个元素的下一个元素是数组的第一个元素），输出每个元素的下一个更大元素。数字 x 的下一个更大的元素是按数组遍历顺序，
 * 这个数字之后的第一个比它更大的数，这意味着你应该循环地搜索它的下一个更大的数。如果不存在，则输出 -1。
 *
 * 思路，一个解法是正常思维解法
 * 还有一个解法是单调栈，
 * 我们首先把第一个元素 A[1] 放入栈，随后对于第二个元素 A[2]，如果 A[2] > A[1]，那么我们就找到了 A[1] 的下一个更大元素 A[2]，此时就可以把 A[1] 出栈并把 A[2] 入栈；
 * 如果 A[2] <= A[1]，我们就仅把 A[2] 入栈。对于第三个元素 A[3]，此时栈中有若干个元素，那么所有比 A[3] 小的元素都找到了下一个更大元素（即 A[3]），
 * 因此可以出栈，在这之后，我们将 A[3] 入栈，以此类推。可以发现，我们维护了一个单调栈，栈中的元素从栈顶到栈底是单调不降的。当我们遇到一个新的元素 A[i] 时，我们判断栈顶元素是否小于 A[i]，如果是，那么栈顶元素的下一个更大元素即为 A[i]
 * 我们将栈顶元素出栈。重复这一操作，直到栈为空或者栈顶元素大于 A[i]。此时我们将 A[i] 入栈，保持栈的单调性，并对接下来的 A[i + 1], A[i + 2] ... 执行同样的操作。
 *
 * 由于这道题的数组是循环数组，因此我们需要将每个元素都入栈两次。这样可能会有元素出栈找过一次，即得到了超过一个“下一个更大元素”，我们只需要保留第一个出栈的结果即可。
 */
public class NextGreaterElementII {

  public static void main(String[] args) {
    int test1[] = {1,2,1};
    int test2[] = {1,5,3,6,8};
    int result[] = new NextGreaterElementII().nextGreaterElements(test2);
    for (int i = 0; i < result.length; i++) {
      System.out.println(result[i]);
    }
  }

  public int[] nextGreaterElements(int[] nums) {
    int[] result = new int[nums.length];
    for (int i = 0; i < nums.length; i++) {
      result[i] = findNum(nums, i, nums[i]);
    }
    return result;

    /**
     * 单调栈方法如下
     */
//    int[] res = new int[nums.length];
//    Stack<Integer> stack = new Stack<>();
//    for (int i = 2 * nums.length - 1; i >= 0; --i) {
//      while (!stack.empty() && nums[stack.peek()] <= nums[i % nums.length]) {
//        stack.pop();
//      }
//      res[i % nums.length] = stack.empty() ? -1 : nums[stack.peek()];
//      stack.push(i % nums.length);
//    }
//    return res;

  }

  private int findNum(int[] array, int num, int value) {
    int result = -1;

    if (num == 0) {
      for (int i = 1; i < array.length; i++) {
        if (array[i] > value) {
          return array[i];
        }
      }
    } else if (num == array.length - 1) {
      for (int i = 0; i < array.length - 1; i++) {
        if (array[i] > value) {
          return array[i];
        }
      }
    } else {
      //前后都要
      for (int i = num + 1; i < array.length; i++) {
        if (array[i] > value) {
          return array[i];
        }
      }
      for (int i = 0; i < num; i++) {
        if (array[i] > value) {
          return array[i];
        }
      }

    }

    return result;
  }

}

```

#### 39.最大二叉树（LC）

题目：给定一个不含重复元素的整数数组。一个以此数组构建的最大二叉树定义如下：1.二叉树的根是数组中的最大元素 2.左子树是通过数组中最大值左边部分构造出的最大二叉树 3.右子树是通过数组中最大值右边部分构造出的最大二叉树，通过给定的数组构建最大二叉树，并且输出这个树的根节点

思路：递归写吧，最明确

```java
package com.qiming.algorithm.leetcode;

/**
 * 最大二叉树
 *
 * 给定一个不含重复元素的整数数组。一个以此数组构建的最大二叉树定义如下：1.二叉树的根是数组中的最大元素 2.左子树是通过数组中最大值左边部分构造出的最大二叉树
 * 3.右子树是通过数组中最大值右边部分构造出的最大二叉树，通过给定的数组构建最大二叉树，并且输出这个树的根节点
 *
 * 思路，递归写吧，最明确
 */
public class MaximumBinaryTree {

  public static void main(String[] args) {

    int[] test = {3,2,1,6,0,5};
    int[] test1 = new int[0];
    System.out.println(test1.length);
  }

  /**
   * 这段可以优化，自己搞下吧
   * @param nums
   * @return
   */
  public TreeNodeMaximumBinaryTree constructMaximumBinaryTree(int[] nums) {
    if (nums.length == 0) {
      return null;
    }

    if (nums.length == 1) {
      return new TreeNodeMaximumBinaryTree(nums[0]);
    }

    int maxIndex = maxIndex(nums, 0, nums.length);
    TreeNodeMaximumBinaryTree root = new TreeNodeMaximumBinaryTree(nums[maxIndex]);
    int left[] = new int[maxIndex - 0];
    for (int i = 0; i < left.length; i++) {
      left[i] = nums[i];
    }
    int right[] = new int[nums.length - maxIndex - 1];
    for (int i = 0; i < right.length; i++) {
      right[i] = nums[maxIndex + i + 1];
    }
    root.left = constructMaximumBinaryTree(left);
    root.right = constructMaximumBinaryTree(right);
    return root;
  }

  private int maxIndex (int[] array, int low, int high) {
    int index = low;
    for (int i = low + 1; i < high; i++) {
      if (array[i] > array[index]) {
        index = i;
      }
    }
    return index;
  }

}

class TreeNodeMaximumBinaryTree {
  int val;
  TreeNodeMaximumBinaryTree left;
  TreeNodeMaximumBinaryTree right;
  TreeNodeMaximumBinaryTree(int x) { val = x; }
}
```

#### 40.最佳买卖股票时机含冷冻期（LC）

题目：给定一个整数数组，其中第 i 个元素代表了第 i 天的股票价格，设计一个算法计算出最大利润。在满足以下约束条件下，你可以尽可能地完成更多的交易（多次买卖一支股票）1. 你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票） 2. 卖出股票后，你无法在第二天买入股票 (即冷冻期为 1 天)

思路：动态规划，不说了  https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/solution/yi-ge-fang-fa-tuan-mie-6-dao-gu-piao-wen-ti-by-lab/

```java
package com.qiming.algorithm.leetcode;

/**
 * 最佳买卖股票时机含冷冻期
 *
 * 给定一个整数数组，其中第 i 个元素代表了第 i 天的股票价格，设计一个算法计算出最大利润。在满足以下约束条件下，你可以尽可能地完成更多的交易（多次买卖一支股票）
 * 1. 你不能同时参与多笔交易（你必须在再次购买前出售掉之前的股票） 2. 卖出股票后，你无法在第二天买入股票 (即冷冻期为 1 天)
 *
 * 思路 动态规划，不说了  https://leetcode-cn.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/solution/yi-ge-fang-fa-tuan-mie-6-dao-gu-piao-wen-ti-by-lab/
 *
 */
public class BestTimeToBuyAndSellStockWithCooldown {

  public int maxProfit(int[] prices) {
    int n = prices.length;
    int dp_i_0 = 0, dp_i_1 = Integer.MIN_VALUE;
    int dp_pre_0 = 0; // 代表 dp[i-2][0]
    for (int i = 0; i < n; i++) {
      int temp = dp_i_0;
      dp_i_0 = Math.max(dp_i_0, dp_i_1 + prices[i]);
      dp_i_1 = Math.max(dp_i_1, dp_pre_0 - prices[i]);
      dp_pre_0 = temp;
    }
    return dp_i_0;
  }

}

```

#### 41.递减元素使数组呈锯齿状（LC）

题目：给你一个整数数组 nums，每次 操作 会从中选择一个元素并 将该元素的值减少 1。如果符合下列情况之一，则数组 A 就是 锯齿数组：每个偶数索引对应的元素都大于相邻的元素，即 A[0] > A[1] < A[2] > A[3] < A[4] > ...或者，每个奇数索引对应的元素都大于相邻的元素，即 A[0] < A[1] > A[2] < A[3] > A[4] < ...返回将数组 nums 转换为锯齿数组所需的最小操作次数。

思路：因为只能降，所以计算两种（奇数位降和偶数位降）分别需要的次数，取小的那个就行了，注意判断好边界

```java
package com.qiming.algorithm.leetcode;

/**
 * 递减元素使数组呈锯齿状
 *
 * 给你一个整数数组 nums，每次 操作 会从中选择一个元素并 将该元素的值减少 1。如果符合下列情况之一，则数组 A 就是 锯齿数组：
 * 每个偶数索引对应的元素都大于相邻的元素，即 A[0] > A[1] < A[2] > A[3] < A[4] > ...
 * 或者，每个奇数索引对应的元素都大于相邻的元素，即 A[0] < A[1] > A[2] < A[3] > A[4] < ...
 * 返回将数组 nums 转换为锯齿数组所需的最小操作次数。
 *
 * 思路 因为只能降，所以计算两种（奇数位降和偶数位降）分别需要的次数，取小的那个就行了，注意判断好边界
 */
public class DecreaseElementsToMakeArrayZigzag {

  public static void main(String[] args) {
//    int[] test = {9,6,1,6,2};
    int[] test1 = {1,2,3};
    System.out.println(new DecreaseElementsToMakeArrayZigzag().movesToMakeZigzag(test1));
  }

  public int movesToMakeZigzag(int[] nums) {

    int len = nums.length;
    int result1 = 0, result2 = 0;
    for (int i = 0; i < len; i++) {
      //偶数位置
      if (i % 2 == 0) {
        int d1,d2;
        if (i > 0 && nums[i] >= nums[i-1]) {
          d1 = nums[i] - nums[i-1] + 1;
        } else {
          d1 = 0;
        }
        if (i < len-1 && nums[i] >= nums[i+1]) {
          d2 = nums[i] - nums[i+1] + 1;
        } else {
          d2 = 0;
        }
        result1 += Math.max(d1, d2);
      } else {
        //奇数位置
        int d1,d2;
        if (i > 0 && nums[i] >= nums[i - 1]) {
          d1 = nums[i] - nums[i - 1] + 1;
        } else {
          d1 = 0;
        }
        if (i < len - 1 && nums[i] >= nums[i + 1]) {
          d2 = nums[i] - nums[i + 1] + 1;
        } else {
          d2 = 0;
        }
        result2 += Math.max(d1, d2);
      }

    }
    return Math.min(result1, result2);
  }

}

```

#### 42.每日温度（LC）

题目：根据每日 气温 列表，请重新生成一个列表，对应位置的输出是你需要再等待多久温度才会升高超过该日的天数。如果之后都不会升高，请在该位置用 0 来代替。例如，给定一个列表 temperatures = [73, 74, 75, 71, 69, 72, 76, 73]，你的输出应该是 [1, 1, 4, 2, 1, 1, 0, 0]。提示：气温 列表长度的范围是 [1, 30000]。每个气温的值的均为华氏度，都是在 [30, 100] 范围内的整数。

思路：用单调栈方式解决，但这边不一样的是入栈的是数组的索引，不是值，方便计算过了多少天，这是关键

```java
package com.qiming.algorithm.leetcode;

import java.util.Stack;

/**
 * 每日温度
 *
 * 根据每日 气温 列表，请重新生成一个列表，对应位置的输出是你需要再等待多久温度才会升高超过该日的天数。如果之后都不会升高，请在该位置用 0 来代替。
 * 例如，给定一个列表 temperatures = [73, 74, 75, 71, 69, 72, 76, 73]，你的输出应该是 [1, 1, 4, 2, 1, 1, 0, 0]。
 *
 * 提示：气温 列表长度的范围是 [1, 30000]。每个气温的值的均为华氏度，都是在 [30, 100] 范围内的整数。
 *
 * 思路，用单调栈方式解决，但这边不一样的是入栈的是数组的索引，不是值，方便计算过了多少天，这是关键
 *
 */
public class DailyTemperatures {

  public static void main(String[] args) {
    int temperatures[] = {73, 74, 75, 71, 69, 72, 76, 73};
    temperatures = new DailyTemperatures().dailyTemperatures(temperatures);
    for (int i = 0; i < temperatures.length; i++) {
      System.out.print(temperatures[i] + " ");
    }
  }

  public int[] dailyTemperatures(int[] T) {
    if (T.length == 1) {
      T[0] = 0;
      return T;
    }

    Stack<Integer> stack = new Stack();
    //这边不一样的地方是把索引push到栈中，而不是值，这样方便计算过后多少天
    stack.push(0);
    for (int i = 1; i < T.length; i++) {
      if (!stack.isEmpty() && T[i] <= T[stack.peek()]) {
        stack.push(i);
      } else {
        //peek在栈空的时候会报错的，pop也是
        while (!stack.isEmpty() && T[stack.peek()] < T[i]) {
          //pop的时候就是遇到了更大值
          int index = stack.pop();
          //pop的点就可以计算过了多少天，是i-pop出来的索引
          T[index] = i - index;
        }
        //压入新值
        stack.push(i);
      }
    }
    //最后栈中所有的索引出来的都是没有更高温度的
    while (!stack.isEmpty()) {
      int index = stack.pop();
      T[index] = 0;
    }

    return T;
  }

}

```

#### 43.旋转链表（LC）

题目：给定一个链表，旋转链表，将链表每个节点向右移动 k 个位置，其中 k 是非负数。输入: 0->1->2->NULL, k = 4  输出: 2->0->1->NULL，解释: 向右旋转 1 步: 2->0->1->NULL  向右旋转 2 步: 1->2->0->NULL  向右旋转 3 步: 0->1->2->NULL  向右旋转 4 步: 2->0->1->NULL

思路：先算一遍长度，然后实际需要转的结点数是 k % size，之后就是细节和链表的一些基本操作

```java
package com.qiming.algorithm.leetcode;

/**
 * 旋转链表
 *
 * 给定一个链表，旋转链表，将链表每个节点向右移动 k 个位置，其中 k 是非负数。
 *
 * 输入: 0->1->2->NULL, k = 4  输出: 2->0->1->NULL
 *
 * 解释: 向右旋转 1 步: 2->0->1->NULL  向右旋转 2 步: 1->2->0->NULL  向右旋转 3 步: 0->1->2->NULL  向右旋转 4 步: 2->0->1->NULL
 *
 * 思路 先算一遍长度，然后实际需要转的结点数是 k % size，之后就是细节和链表的一些基本操作
 */
public class RotateList {

  public static void main(String[] args) {
    ListNodeRotateList test1 = new ListNodeRotateList(0);
    ListNodeRotateList test2 = new ListNodeRotateList(1);
    ListNodeRotateList test3 = new ListNodeRotateList(2);
    test1.next = test2;
    test2.next = test3;
    test3.next =null;
    ListNodeRotateList result = new RotateList().rotateRight(test1, 4);

  }

  public ListNodeRotateList rotateRight(ListNodeRotateList head, int k) {
    ListNodeRotateList p = head;
    //算长度
    int size = 0;
    while (p != null) {
      size++;
      p = p.next;
    }
    if (size == 0) {
      return head;
    } else {
      int realTime = k % size;
      if (realTime == 0) {
        //不用变
        return head;
      } else {
        //变最后realTime个结点，移到前面即可
        ListNodeRotateList j = head;
        for (int i = 0; i < size - realTime - 1; i++) {
          j = j.next;
        }
        ListNodeRotateList newHead = j.next;
        //置为尾结点
        j.next = null;
        ListNodeRotateList tmp = newHead;
        for (int i = 0; i < realTime - 1; i++) {
          tmp = tmp.next;
        }
        tmp.next = head;
        return newHead;
      }
    }
  }

}


class ListNodeRotateList{
  int val;
  ListNodeRotateList next;
  ListNodeRotateList(int x) { val = x; }
}
```
#### 44.自定义字符串排序（LC）

题目：字符串S和 T 只包含小写字符。在S中，所有字符只会出现一次。S 已经根据某种规则进行了排序。我们要根据S中的字符顺序对T进行排序。更具体地说，如果S中x在y之前出现，那么返回的字符串中x也应出现在y之前。返回任意一种符合条件的字符串T。输入: S = "cba"  T = "abcd"   输出: "cbad"，解释: S中出现了字符 "a", "b", "c", 所以 "a", "b", "c" 的顺序应该是 "c", "b", "a". 由于 "d" 没有在S中出现, 它可以放在T的任意位置. "dcba", "cdba", "cbda" 都是合法的输出。注意: S的最大长度为26，其中没有重复的字符。 T的最大长度为200。 S和T只包含小写字符。

思路：个人思路，用linkedhashmap，即排序，又有字符的key和出现的次数，然后剩余没有的往char的数组后面放就行了。另外 StringBuilder可以append字符，然后再toString()转成字符串

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedHashMap;

/**
 * 自定义字符串排序
 *
 * 字符串S和 T 只包含小写字符。在S中，所有字符只会出现一次。S 已经根据某种规则进行了排序。
   * 我们要根据S中的字符顺序对T进行排序。更具体地说，如果S中x在y之前出现，那么返回的字符串中x也应出现在y之前。返回任意一种符合条件的字符串T。
 *
 * 输入: S = "cba"  T = "abcd"   输出: "cbad"
 *
 * 解释: S中出现了字符 "a", "b", "c", 所以 "a", "b", "c" 的顺序应该是 "c", "b", "a". 由于 "d" 没有在S中出现, 它可以放在T的任意位置. "dcba", "cdba", "cbda" 都是合法的输出。
 *
 * 注意: S的最大长度为26，其中没有重复的字符。 T的最大长度为200。 S和T只包含小写字符。
 *
 * 思路 个人思路，用linkedhashmap，即排序，又有字符的key和出现的次数，然后剩余没有的往char的数组后面放就行了
 *
 * 另外 StringBuilder可以append字符，然后再toString()转成字符串
 *
 */
public class CustomSortString {

  public static void main(String[] args) {

    String S = "cba";
    String T = "abcd";

    System.out.println(new CustomSortString().customSortString(S, T));

  }

  public String customSortString(String S, String T) {

    if (S == null || S.length() == 0) {
      return T;
    }

    if (T == null) {
      return null;
    }

    //key和次数，自带顺序
    LinkedHashMap<Character, Integer> map = new LinkedHashMap();
    for (int i = 0; i < S.length(); i++) {
      char a = S.charAt(i);
      map.put(a, 0);
    }

    int len = T.length();
    char[] chars = new char[len];
    int right = T.length() - 1;
    for (int i = 0; i < len; i++) {
      char tmp = T.charAt(i);
      if (map.containsKey(tmp)) {
        //说明有这个值，次数加1
        map.put(tmp, map.get(tmp) + 1);
      } else {
        //说明没有这个值，放到结尾
        chars[right] = tmp;
        right--;
      }
    }
    int begin = 0;
    for (Character character : map.keySet()) {
      int size = map.get(character);
      for (int i = begin; i < begin + size; i++) {
        chars[i] = character;
      }
      begin = begin + size;
    }
    return String.valueOf(chars);
  }

}

```

#### 45.螺旋矩阵（LC）

题目：给定一个包含 m x n 个元素的矩阵（m 行, n 列），请按照顺时针螺旋顺序，返回矩阵中的所有元素。

思路：就是生转，这边主要是建立该什么时候转跟怎么转

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.List;

/**
 * 螺旋矩阵
 *
 * 给定一个包含 m x n 个元素的矩阵（m 行, n 列），请按照顺时针螺旋顺序，返回矩阵中的所有元素。
 *
 * 输入:
 * [
 *  [ 1, 2, 3 ],
 *  [ 4, 5, 6 ],
 *  [ 7, 8, 9 ]
 * ]
 *
 * 输出: [1,2,3,6,9,8,7,4,5]
 *
 * 思路 就是生转，这边主要是建立该什么时候转跟怎么转
 *
 */
public class SpiralMatrix {

  public List<Integer> spiralOrder(int[][] matrix) {
    List ans = new LinkedList();
    if (matrix.length == 0) return ans;
    int R = matrix.length, C = matrix[0].length;
    boolean[][] seen = new boolean[R][C];
    //下面两个其实定义死了方向，两个数组的0标意思是向右，1标意思是向下，2标是向走，3标识向上
    int[] dr = {0, 1, 0, -1};
    int[] dc = {1, 0, -1, 0};
    int r = 0, c = 0, di = 0;
    for (int i = 0; i < R * C; i++) {
      ans.add(matrix[r][c]);
      seen[r][c] = true;
      int cr = r + dr[di];
      int cc = c + dc[di];
      //不符合就顺时针转，条件如下
      if (0 <= cr && cr < R && 0 <= cc && cc < C && !seen[cr][cc]){
        r = cr;
        c = cc;
      } else {
        //用mod转
        di = (di + 1) % 4;
        r += dr[di];
        c += dc[di];
      }
    }
    return ans;

  }

}

```
#### 46.替换后的最长重复字符（LC）

题目：给你一个仅由大写英文字母组成的字符串，你可以将任意位置上的字符替换成另外的字符，总共可最多替换 k 次。在执行上述操作后，找到包含重复字母的最长子串的长度。字符串长度 和 k 不会超过 10^4。输入:s = "AABABBA", k = 1 输出: 4

思路：滑动窗口，和之前的LongestStringLength一个道理

```java
package com.qiming.algorithm.leetcode;

/**
 * 替换后的最长重复字符
 *
 * 给你一个仅由大写英文字母组成的字符串，你可以将任意位置上的字符替换成另外的字符，总共可最多替换 k 次。在执行上述操作后，找到包含重复字母的最长子串的长度。
 * 字符串长度 和 k 不会超过 10^4。输入:s = "AABABBA", k = 1 输出: 4
 *
 * 思路 滑动窗口，和之前的LongestStringLength一个道理
 *
 * 当K>0时，子串的条件变成了允许我们变换子串中的K个字符使其变成一个连续子串，那么这个题的关键点就是我们如何判断一个字符串改变K个字符，能够变成一个连续串
 *
 * 如果当前字符串中的出现次数最多的字母个数+K大于串长度，那么这个串就是满足条件的
 *
 * 我们维护一个数组int[26]来存储当前窗口中各个字母的出现次数（注意当前窗口这个说法很重要），left表示窗口的左边界，right表示窗口右边界，窗口扩张：left不变，right++，窗口滑动：left++, right++
 *
 * charMax保存滑动窗口内相同字母出现次数的历史最大值，通过判断窗口宽度(right - left + 1)是否大于charMax + K来决定窗口是否做滑动，否则窗口就扩张，这个很关键
 *
 *
 */
public class LongestRepeatingCharacterReplacement {

  private int[] map = new int[26];

  public int characterReplacement(String s, int k) {
    if (s == null) {
      return 0;
    }
    char[] chars = s.toCharArray();
    int left = 0;
    int right = 0;
    int charMax = 0;
    for (right = 0; right < chars.length; right++) {
      int index = chars[right] - 'A';
      map[index]++;
      charMax = Math.max(charMax, map[index]);
      if (right - left + 1 > charMax + k) {
        //滑动，注意是当前窗口，所以有个map--
        map[chars[left] - 'A']--;
        left++;
      }
      //否则就是扩张
    }
    //最后就是返回窗口的长度
    return chars.length - left;

  }

}

```

#### 47.分割数组（LC）

题目：给定一个数组 A，将其划分为两个不相交（没有公共元素）的连续子数组 left 和 right， 使得：1、left 中的每个元素都小于或等于 right 中的每个元素。2、left 和 right 都是非空的。3、left 要尽可能小。在完成这样的分组后返回 left 的长度。可以保证存在这样的划分方法。

思路：自己的想法，效果不是太好，就是左边最大值和右边最小值，不停更新，直到leftMax <= rightMin。比较好解法在思路上是一样的，但是，用到了两个辅助数组，分别记录在某个位置上该有的最大值和最小值，再进行比较，但是好像效果也不怎么样。。

```java
package com.qiming.algorithm.leetcode;

/**
 * 分割数组
 *
 * 给定一个数组 A，将其划分为两个不相交（没有公共元素）的连续子数组 left 和 right， 使得：
 * 1、left 中的每个元素都小于或等于 right 中的每个元素。2、left 和 right 都是非空的。3、left 要尽可能小。
 * 在完成这样的分组后返回 left 的长度。可以保证存在这样的划分方法。
 *
 * 输入：[1,1,1,0,6,12] 输出：4 解释：left = [1,1,1,0]，right = [6,12] 可以保证至少有一种方法能够按题目所描述的那样对 A 进行划分。
 *
 * 思路，自己的想法，效果不是太好，就是左边最大值和右边最小值，不停更新，直到leftMax <= rightMin
 *
 * 比较好解法在思路上是一样的，但是，用到了两个辅助数组，分别记录在某个位置上该有的最大值和最小值，再进行比较，但是好像效果也不怎么样。。
 */
public class PartitionArrayIntoDisjointIntervals {

  public static void main(String[] args) {
    int test1[] = {5,0,3,8,6};
    int test2[] = {1,1,1,0,6,12};
    System.out.println(new PartitionArrayIntoDisjointIntervals().partitionDisjoint(test2));
  }

  public int partitionDisjoint(int[] A) {
    /**
     * 自己的思路
     */
    //    if (A.length == 2) {
//      return 1;
//    }
//    int leftMax = A[0];
//    //找right的最小值
//    int rightMin = 1_000_000;
//    for (int i = 1; i < A.length; i++) {
//      if (A[i] < rightMin) {
//        rightMin = A[i];
//      }
//    }
//    for (int i = 0; i < A.length; i++) {
//      if (leftMax <= rightMin) {
//        return i + 1;
//      }
//      leftMax = Math.max(leftMax, A[i+1]);
//      if (A[i+1] == rightMin) {
//        //需要处理一下了，可能有重复值，判断是不是还是rightMin了
//        int newRightMin = A[i+2];
//        for (int j = i + 3; j < A.length; j++) {
//          if (A[j] < newRightMin) {
//            newRightMin = A[j];
//          }
//        }
//        rightMin = newRightMin == rightMin ? rightMin : newRightMin;
//      }
//      //否则rightMin不懂
//    }
//    return -1;

    /**
     * 辅助数组的思路
     */
    int N = A.length;
    int[] maxleft = new int[N];
    int[] minright = new int[N];

    int m = A[0];
    for (int i = 0; i < N; ++i) {
      m = Math.max(m, A[i]);
      maxleft[i] = m;
    }

    m = A[N-1];
    for (int i = N-1; i >= 0; --i) {
      m = Math.min(m, A[i]);
      minright[i] = m;
    }

    for (int i = 1; i < N; ++i)
      if (maxleft[i-1] <= minright[i])
        return i;

    return -1;
  }

}

```

#### 48.字母异位词分组（LC）

题目：给定一个字符串数组，将字母异位词组合在一起。字母异位词指字母相同，但排列不同的字符串。输入: ["eat", "tea", "tan", "ate", "nat", "bat"],输出:
 * [
 *   ["ate","eat","tea"],
 *   ["nat","tan"],
 *   ["bat"]
 * ]

思路：自己的思路，用小写字母的出现+次数的拼接作为map的key，然后值就是存储的String。其他思路 当且仅当它们的排序字符串相等时，两个字符串是字母异位词

```java
package com.qiming.algorithm.leetcode;

import java.util.HashMap;
import java.util.LinkedList;
import java.util.List;

/**
 * 字母异位词分组
 *
 * 给定一个字符串数组，将字母异位词组合在一起。字母异位词指字母相同，但排列不同的字符串。
 *
 * 输入: ["eat", "tea", "tan", "ate", "nat", "bat"],
 *
 * 输出:
 * [
 *   ["ate","eat","tea"],
 *   ["nat","tan"],
 *   ["bat"]
 * ]
 *
 * 说明： 所有输入均为小写字母。不考虑答案输出的顺序。
 *
 * 思路 自己的思路，用小写字母的出现+次数的拼接作为map的key，然后值就是存储的String
 *
 * 其他思路 当且仅当它们的排序字符串相等时，两个字符串是字母异位词
 *
 */
public class GroupAnagrams {

  public static void main(String[] args) {
    String[] s = {"eat", "tea", "tan", "ate", "nat", "bat"};
    List<List<String>> result = new GroupAnagrams().groupAnagrams(s);
    for (List<String> strings : result) {
      for (String string : strings) {
        System.out.print(string + " ");
      }
      System.out.println("\n");
    }
  }

  public List<List<String>> groupAnagrams(String[] strs) {

    List<List<String>> result = new LinkedList<>();

    if (strs.length == 0) {
      return result;
    }
    HashMap<String, List<String>> map = new HashMap<>();

    for (int i = 0; i < strs.length; i++) {
      String s = strs[i];
      char[] chars = s.toCharArray();
      int[] charCount = new int[26];
      for (int j = 0; j < chars.length; j++) {
        charCount[chars[j] - 'a']++;
      }
      StringBuilder sb = new StringBuilder();
      for (int j = 0; j < charCount.length; j++) {
        if (charCount[j] != 0) {
          //说明有数量
          sb.append(j);
          sb.append('_');
          sb.append(charCount[j]);
          sb.append('_');
        }
      }
      String key = sb.toString();
      if (map.containsKey(key)) {
        List<String> oldList = map.get(key);
        oldList.add(s);
      } else {
        List<String> newList = new LinkedList<>();
        newList.add(s);
        map.put(key, newList);
      }
    }

    for (String s : map.keySet()) {
      result.add(map.get(s));
    }
    return result;

//    当且仅当它们的排序字符串相等时，两个字符串是字母异位词
//    if (strs.length == 0) return new ArrayList();
//    Map<String, List> ans = new HashMap<String, List>();
//    for (String s : strs) {
//      char[] ca = s.toCharArray();
//      Arrays.sort(ca);
//      String key = String.valueOf(ca);
//      if (!ans.containsKey(key)) ans.put(key, new ArrayList());
//      ans.get(key).add(s);
//    }
//    return new ArrayList(ans.values());

  }

}

```

#### 49.匹配子序列的单词数（LC）

题目：给定字符串 S 和单词字典 words, 求 words[i] 中是 S 的子序列的单词个数。输入: S = "abcde" words = ["a", "bb", "acd", "ace"] 输出: 3。解释: 有三个是 S 的子序列的单词: "a", "acd", "ace"。

思路：构建一种桶的结构，把桶装在26个字母中，桶的构造是根据words中词的单词的首字母构造再不停变换

```java
package com.qiming.algorithm.leetcode;

import java.util.ArrayList;

/**
 * 匹配子序列的单词数
 *
 * 给定字符串 S 和单词字典 words, 求 words[i] 中是 S 的子序列的单词个数。输入: S = "abcde" words = ["a", "bb", "acd", "ace"] 输出: 3
 * 解释: 有三个是 S 的子序列的单词: "a", "acd", "ace"。
 *
 * 注意: 所有在words和 S 里的单词都只由小写字母组成。S 的长度在 [1, 50000]。words 的长度在 [1, 5000]。words[i]的长度在[1, 50]。
 *
 * 思路，构建一种桶的结构，把桶装在26个字母中，桶的构造是根据words中词的单词的首字母构造再不停变换
 *
 */
public class NumberOfMatchingSubsequences {

  public int numMatchingSubseq(String S, String[] words) {

    int ans = 0;
    ArrayList<Node>[] heads = new ArrayList[26];
    for (int i = 0; i < 26; ++i)
      heads[i] = new ArrayList<Node>();

    for (String word: words)
      heads[word.charAt(0) - 'a'].add(new Node(word, 0));

    for (char c: S.toCharArray()) {
      ArrayList<Node> old_bucket = heads[c - 'a'];
      //重置一个list的node
      heads[c - 'a'] = new ArrayList<Node>();

      for (Node node: old_bucket) {
        node.index++;
        if (node.index == node.word.length()) {
          ans++;
        } else {
          heads[node.word.charAt(node.index) - 'a'].add(node);
        }
      }
      old_bucket.clear();
    }
    return ans;

  }

}
class Node {
  String word;
  //用于判断最后的机构
  int index;
  public Node(String w, int i) {
    word = w;
    index = i;
  }
}
```

#### 50.扁平化多级双向链表（LC）

题目：您将获得一个双向链表，除了下一个和前一个指针之外，它还有一个子指针，可能指向单独的双向链表。这些子列表可能有一个或多个自己的子项，依此类推，生成多级数据结构，如下面的示例所示。扁平化列表，使所有结点出现在单级双链表中。您将获得列表第一级的头部。* 输入:
 *  1---2---3---4---5---6--NULL
 *          |
 *          7---8---9---10--NULL
 *              |
 *              11--12--NULL
 *
 * 输出:
 * 1-2-3-7-8-11-12-9-10-4-5-6-NULL

思路：递归的思路，注意判空跟置child为空，细节细节

```java
package com.qiming.algorithm.leetcode;

/**
 * 扁平化多级双向链表
 *
 * 您将获得一个双向链表，除了下一个和前一个指针之外，它还有一个子指针，可能指向单独的双向链表。这些子列表可能有一个或多个自己的子项，依此类推，生成多级数据结构，如下面的示例所示。
 * 扁平化列表，使所有结点出现在单级双链表中。您将获得列表第一级的头部。
 *
 * 输入:
 *  1---2---3---4---5---6--NULL
 *          |
 *          7---8---9---10--NULL
 *              |
 *              11--12--NULL
 *
 * 输出:
 * 1-2-3-7-8-11-12-9-10-4-5-6-NULL
 *
 * 思路 递归的思路，注意判空跟置child为空，细节细节
 *
 */
public class FlattenAMultilevelDoublyLinkedList {

  public static void main(String[] args) {
    NodeFlattenAMultilevelDoublyLinkedList node1 = new NodeFlattenAMultilevelDoublyLinkedList();
    node1.val = 1;
    NodeFlattenAMultilevelDoublyLinkedList node2 = new NodeFlattenAMultilevelDoublyLinkedList();
    node2.val = 2;
    NodeFlattenAMultilevelDoublyLinkedList node3 = new NodeFlattenAMultilevelDoublyLinkedList();
    node3.val = 3;
    NodeFlattenAMultilevelDoublyLinkedList node4 = new NodeFlattenAMultilevelDoublyLinkedList();
    node4.val = 4;
    NodeFlattenAMultilevelDoublyLinkedList node5 = new NodeFlattenAMultilevelDoublyLinkedList();
    node5.val = 5;
    NodeFlattenAMultilevelDoublyLinkedList node6 = new NodeFlattenAMultilevelDoublyLinkedList();
    node6.val = 6;
    NodeFlattenAMultilevelDoublyLinkedList node7 = new NodeFlattenAMultilevelDoublyLinkedList();
    node7.val = 7;
    NodeFlattenAMultilevelDoublyLinkedList node8 = new NodeFlattenAMultilevelDoublyLinkedList();
    node8.val = 8;
    NodeFlattenAMultilevelDoublyLinkedList node9 = new NodeFlattenAMultilevelDoublyLinkedList();
    node9.val = 9;
    NodeFlattenAMultilevelDoublyLinkedList node10 = new NodeFlattenAMultilevelDoublyLinkedList();
    node10.val = 10;
    NodeFlattenAMultilevelDoublyLinkedList node11 = new NodeFlattenAMultilevelDoublyLinkedList();
    node11.val = 11;
    NodeFlattenAMultilevelDoublyLinkedList node12 = new NodeFlattenAMultilevelDoublyLinkedList();
    node12.val = 12;

    node1.next = node2;
    node2.prev = node1;
    node2.next = node3;
    node3.prev = node2;
    node3.next = node4;
    node3.child = node7;
    node4.prev = node3;
    node4.next = node5;
    node5.prev = node4;
    node5.next = node6;
    node6.prev = node5;
    node6.next = null;
    node7.next = node8;
    node8.prev = node7;
    node8.next = node9;
    node8.child = node11;
    node9.prev = node8;
    node9.next = node10;
    node10.prev = node9;
    node10.next = null;
    node11.next = node12;
    node12.prev = node11;
    node12.next =null;

    NodeFlattenAMultilevelDoublyLinkedList result = new FlattenAMultilevelDoublyLinkedList().flatten(node1);
    System.out.println("end");
  }

  public NodeFlattenAMultilevelDoublyLinkedList flatten(NodeFlattenAMultilevelDoublyLinkedList head) {
    NodeFlattenAMultilevelDoublyLinkedList p = head;
    while (p != null && p.child == null) {
      p = p.next;
    }
    if (p != null) {
      //说明有child
      NodeFlattenAMultilevelDoublyLinkedList next = p.next;
      p.next = p.child;
      p.child.prev = p;
      NodeFlattenAMultilevelDoublyLinkedList nextLevel = p.child;
      while (nextLevel.next != null) {
        nextLevel = nextLevel.next;
      }
      nextLevel.next = next;
      //nullpoint，有一种特殊的情况考虑，即每一层都只有一个节点，又都有孩子，这时候next就是null了
//      next.prev = nextLevel;
      if (next != null) {
        next.prev = nextLevel;
      }
      flatten(p.child);
      //要置空哦
      p.child = null;
    } else {
      return head;
    }
    return head;
  }

}

class NodeFlattenAMultilevelDoublyLinkedList {
  public int val;
  public NodeFlattenAMultilevelDoublyLinkedList prev;
  public NodeFlattenAMultilevelDoublyLinkedList next;
  public NodeFlattenAMultilevelDoublyLinkedList child;
}
```

#### 51.括号的分数（LC）

题目：给定一个平衡括号字符串 S，按下述规则计算该字符串的分数：1、() 得 1 分。2、AB 得 A + B 分，其中 A 和 B 是平衡括号字符串。3、(A) 得 2 * A 分，其中 A 是平衡括号字符串。输入： "()" 输出： 1  输入： "(())" 输出： 2  输入： "()()"  输出： 2  输入： "(()(()))"  输出： 6

思路：这个思路有点仙了，看代码吧

```java
package com.qiming.algorithm.leetcode;

import java.util.Stack;

/**
 * 括号的分数
 *
 * 给定一个平衡括号字符串 S，按下述规则计算该字符串的分数：1、() 得 1 分。2、AB 得 A + B 分，其中 A 和 B 是平衡括号字符串。3、(A) 得 2 * A 分，其中 A 是平衡括号字符串。
 *
 * 输入： "()" 输出： 1  输入： "(())" 输出： 2  输入： "()()"  输出： 2  输入： "(()(()))"  输出： 6
 *
 * 提示：1、S 是平衡括号字符串，且只含有 ( 和 ) 。2、2 <= S.length <= 50
 *
 * 思路 立马想到的是用栈。但是这个题目怎么用
 *
 * 字符串 S 中的每一个位置都有一个“深度”，即该位置外侧嵌套的括号数目。例如，字符串 (()(.())) 中的 . 的深度为 2，因为它外侧嵌套了 2 层括号：(__(.__))。
 * 我们用一个栈来维护当前所在的深度，以及每一层深度的得分。当我们遇到一个左括号 ( 时，我们将深度加一并且新的深度的得分置为 0。
 * 当我们遇到一个右括号 ) 时，我们将当前深度的得分乘二并加到上一层的深度得分。这里有一种例外情况，如果遇到的是 ()，那么只将得分加一
 *
 * "(()(()))"来做一个示例
 *
 * [0, 0] (
 * [0, 0, 0] ((
 * [0, 1] (()
 * [0, 1, 0] (()(
 * [0, 1, 0, 0] (()((
 * [0, 1, 1] (()(()
 * [0, 3] (()(())
 * [6] (()(()))
 *
 */
public class ScoreOfParentheses {

  public int scoreOfParentheses(String S) {

    Stack<Integer> stack = new Stack();
    stack.push(0); // The score of the current frame

    for (char c: S.toCharArray()) {
      if (c == '(')
        stack.push(0);
      else {
        //当前深度的得分
        int v = stack.pop();
        //上一层的深度得分
        int w = stack.pop();
        stack.push(w + Math.max(2 * v, 1));
      }
    }

    return stack.pop();

  }

}

```

#### 52.合并石头的最低成本（LC）

题目：有 N 堆石头排成一排，第 i 堆中有 stones[i] 块石头。每次移动（move）需要将连续的 K 堆石头合并为一堆，而这个移动的成本为这 K 堆石头的总数。找出把所有石头合并成一堆的最低成本。如果不可能，返回 -1 。

思路：动态规划及其优化

[ref](https://zxi.mytechroad.com/blog/dynamic-programming/leetcode-1000-minimum-cost-to-merge-stones/)

```java
package com.qiming.algorithm.leetcode;

/**
 * 合并石头的最低成本
 *
 * 有 N 堆石头排成一排，第 i 堆中有 stones[i] 块石头。每次移动（move）需要将连续的 K 堆石头合并为一堆，而这个移动的成本为这 K 堆石头的总数。
 * 找出把所有石头合并成一堆的最低成本。如果不可能，返回 -1 。
 *
 * 输入：stones = [3,2,4,1], K = 3 输出：-1
 *
 * 输入：stones = [3,5,1,2,6], K = 3 输出：25
 *
 * Impossible to do search, Greedy doesn't work either. DP is the only approach
 * Non-overlapping subproblems  dp[i][j][k] := min cost to merge A[i] - A[j] into k piles
 * Init: dp[i][i][1] = 0 # no cost to merge one into one
 *
 * dp[i][j][k] = min(dp[i][m][1] + dp[m+1][k-1]), i <= m < j, 2 <=k <=K
 * dp[i][j][1] = dp[i][j][k] + sum(A[i]~A[j])
 *
 * ans: dp[0][n-1][1] merge the whole array into one
 */

public class MinimumCostToMergeStones {

  public static void main(String[] args) {
    System.out.println(-1 % 2);
  }

  public int mergeStones(int[] stones, int K) {
    int size = stones.length;
    int mod = (size - K) % (K-1);
    int maxInt = Integer.MAX_VALUE;
    if (mod != 0) {
      return -1;
    }

    //对应sum(A[i]~A[j]) sums[1]表示到从1元素大1元素，sums[2]表示从1元素到2元素的和
    int []sums = new int[size+1];
    for (int i = 0; i < size; i++) {
      sums[i + 1] = sums[i] + stones[i];
    }
    //dp表初始化为最大值，因为要求的是min
    int dp[][][] = new int[size][size][K+1];
    for (int i = 0; i < dp.length; i++) {
      for (int j = 0; j < dp[i].length; j++) {
        for (int k = 0; k < dp[i][j].length; k++) {
          dp[i][j][k] = maxInt;
        }
      }
    }

    //这是上面的init
    for (int i = 0; i < size; i++) {
      dp[i][i][1] = 0;
    }

    //转换动态方程到代码
    for (int i = 2; i <= size; i++) {  //subproblem length 相当于限制，不能高于这个数了
      for (int j = 0; j <= size - i; j++) {  //start
        int m = j + i - 1; //end
        for (int k = 2; k <= K ; k++) { //piles
          for (int n = j; n < m; n += K-1) {  //split point
            dp[j][m][k] = Math.min(dp[j][m][k], dp[j][n][1] + dp[n+1][m][k-1]);
          }
        }
        dp[j][m][1] = dp[j][m][K] + sums[m+1] - sums[j]; //根据前面的意思这边就减一下，就是sum(A[j]~A[m])
      }
    }
    return dp[0][size-1][1];
  }

}
```

#### 53.二叉树的高度

题目：二叉树的高度

思路：递归和非递归

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 求二叉树的高度
 */
public class DepthOfTree {

  public static void main(String[] args) {
    TreeNodeDepthOfTree t1 = new TreeNodeDepthOfTree(1);
    TreeNodeDepthOfTree t2 = new TreeNodeDepthOfTree(1);
    TreeNodeDepthOfTree t3 = new TreeNodeDepthOfTree(1);
    TreeNodeDepthOfTree t4 = new TreeNodeDepthOfTree(1);
    TreeNodeDepthOfTree t5 = new TreeNodeDepthOfTree(1);
    TreeNodeDepthOfTree t6 = new TreeNodeDepthOfTree(1);
    t1.left = t2;
    t1.right = t3;
    t2.left = t4;
    t4.right = t5;
    t5.left = t6;
    System.out.println(new DepthOfTree().depthByIteration(t1));
    System.out.println(new DepthOfTree().depthByRecursion(t1));
  }

  /**
   * 递归求高度
   * @param root
   * @return
   */
  public int depthByRecursion(TreeNodeDepthOfTree root) {
    if (root == null) {
      return 0;
    }

    int leftDepth = depthByRecursion(root.left) + 1;
    int rightDepth = depthByRecursion(root.right) + 1;

    return leftDepth > rightDepth ? leftDepth : rightDepth;

  }

  /**
   * 非递归求高度，层次遍历
   * @param root
   * @return
   */
  public int depthByIteration(TreeNodeDepthOfTree root) {
    if (root == null) {
      return 0;
    }
    Queue<TreeNodeDepthOfTree> queue = new LinkedList();
    queue.offer(root);
    int height = 0;
    while (!queue.isEmpty()) {
      int size = queue.size();
      for (int i = 0; i < size; i++) {
        TreeNodeDepthOfTree p = queue.poll();
        if (p.left != null) {
          queue.offer(p.left);
        }
        if (p.right != null) {
          queue.offer(p.right);
        }
      }
      height++;
    }
    return height;
  }

}

class TreeNodeDepthOfTree {

  int val;
  TreeNodeDepthOfTree left;
  TreeNodeDepthOfTree right;
  public TreeNodeDepthOfTree(int val) {
    this.val = val;
  }
}
```

#### 54.监控二叉树（LC）

题目：给定一个二叉树，我们在树的节点上安装摄像头。节点上的每个摄影头都可以监视其父对象、自身及其直接子对象。计算监控树的所有节点所需的最小摄像头数量。输入：[0,0,null,0,0]  输出：1

思路：1：该节点安装了监视器 2：该节点可观，但没有安装监视器 3：该节点不可观

```java
package com.qiming.algorithm.leetcode;

/**
 * 监控二叉树
 *
 * 给定一个二叉树，我们在树的节点上安装摄像头。节点上的每个摄影头都可以监视其父对象、自身及其直接子对象。计算监控树的所有节点所需的最小摄像头数量。
 *
 * 输入：[0,0,null,0,0]  输出：1
 *
 * 提示：给定树的节点数的范围是 [1, 1000]。 每个节点的值都是 0。
 *
 * 思路：1：该节点安装了监视器 2：该节点可观，但没有安装监视器 3：该节点不可观
 *
 */
public class BinaryTreeCameras {

  private int ans = 0;

  public int minCameraCover(TreeNode root) {
    if (root == null) return 0;
    if (dfs(root) == 2) ans++;
    return ans;
  }

  // 1：该节点安装了监视器 2：该节点可观，但没有安装监视器 3：该节点不可观
  private int dfs(TreeNode node) {
    if (node == null)
      return 1;
    int left = dfs(node.left), right = dfs(node.right);
    if (left == 2 || right == 2) {
      ans++;
      return 0;
    } else if (left == 0 || right == 0){
      return 1;
    } else
      return 2;
  }

}

```


#### 55.子集（LC）

题目：给定一组不含重复元素的整数数组 nums，返回该数组所有可能的子集（幂集）。说明：解集不能包含重复的子集。给定一组不含重复元素的整数数组 nums，返回该数组所有可能的子集（幂集）。说明：解集不能包含重复的子集。

思路：回溯算法，模板解，无终止条件

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.List;

/**
 * 子集
 *
 * 给定一组不含重复元素的整数数组 nums，返回该数组所有可能的子集（幂集）。说明：解集不能包含重复的子集。
 *
 * 输入: nums = [1,2,3]  输出: [[3],[1],[2],[1,2,3],[1,3],[2,3],[1,2],[]]
 *
 * 思路：回溯算法，模板解，无终止条件
 *
 */
public class Subsets {

  private List<List<Integer>> res = new LinkedList<>();

  public static void main(String[] args) {
    int nums[] = new int[3];
    for (int i = 0; i < nums.length; i++) {
      nums[i] = i + 1;
    }
    List<List<Integer>> result = new Subsets().subsets(nums);
    for (List<Integer> integers : result) {
      for (Integer integer : integers) {
        System.out.print(integer + " ");
      }
      System.out.println("\n");
    }
  }

  public List<List<Integer>> subsets(int[] nums) {

    LinkedList<Integer> track = new LinkedList<>();
    backtrack(nums, track, 0);
    return res;
  }

  private void backtrack(int nums[], LinkedList<Integer> track, int begin) {
    //这个题目就是没有终止条件，直接运行完就行，注意这个begin和new出来的track，否则res里面会出现很诡异的同值
    res.add(new LinkedList<>(track));
    for (int i = begin; i < nums.length; i++) {
      track.add(nums[i]);
      backtrack(nums, track, i + 1);
      //取消选择
      track.removeLast();
    }
  }

}

```

#### 56.有序队列（LC）

题目：给出了一个由小写字母组成的字符串 S。然后，我们可以进行任意次数的移动。在每次移动中，我们选择前 K 个字母中的一个（从左侧开始），将其从原位置移除，并放置在字符串的末尾。返回我们在任意次数的移动之后可以拥有的按字典顺序排列的最小字符串。

思路：数学思想，其实只有k=1和k=2两种情况

```java
package com.qiming.algorithm.leetcode;

import java.util.Arrays;

/**
 * 有序队列
 *
 * 给出了一个由小写字母组成的字符串 S。然后，我们可以进行任意次数的移动。在每次移动中，我们选择前 K 个字母中的一个（从左侧开始），将其从原位置移除，并放置在字符串的末尾。
 * 返回我们在任意次数的移动之后可以拥有的按字典顺序排列的最小字符串。
 *
 * 提示：1、 1 <= K <= S.length <= 1000  2、 S 只由小写字母组成。
 *
 *
 * 思路：当 K = 1 时，每次操作只能将第一个字符移动到末尾，因此字符串 S 可以看成一个头尾相连的环。如果 S 的长度为 NN，我们只需要找出这 NN 个位置中字典序最小的字符串即可。
 * 当 K = 2 时，可以发现，我们能够交换字符串中任意两个相邻的字母。具体地，设字符串 S 为 S[1], S[2], ..., S[i], S[i + 1], ..., S[N]，我们需要交换 S[i] 和 S[j]。
 * 首先我们依次将 S[i] 之前的所有字符依次移到末尾，得到S[i], S[i + 1], ..., S[N], S[1], S[2], ..., S[i - 1]，随后我们先将 S[i + 1] 移到末尾，再将 S[i] 移到末尾，得到
 * S[i + 2], ..., S[N], S[1], S[2], ..., S[i - 1], S[i + 1], S[i]，最后将 S[i + 1] 之后的所有字符依次移到末尾，得到
 * S[1], S[2], ..., S[i - 1], S[i + 1], S[i], S[i + 2], ..., S[N]，这样就交换了 S[i] 和 S[i + 1]，而没有改变其余字符的位置。
 * 当我们可以交换任意两个相邻的字母后，就可以使用冒泡排序的方法，仅通过交换相邻两个字母，使得字符串变得有序。因此当 K = 2 时，我们可以将字符串移动得到最小的字典序。
 * 当 K > 2 时，我们可以完成 K = 2 时的所有操作。
 *
 *
 */
public class OrderlyQueue {

  public static void main(String[] args) {
    String s = "baaca";
    System.out.println(new OrderlyQueue().orderlyQueue(s, 1));
  }

  public String orderlyQueue(String S, int K) {
    if (K == 1) {
      String ans = S;
      for (int i = 0; i < S.length(); ++i) {
        String T = S.substring(i) + S.substring(0, i);
        if (T.compareTo(ans) < 0) ans = T;
      }
      return ans;
    } else {
      char[] ca = S.toCharArray();
      Arrays.sort(ca);
      return new String(ca);
    }
  }

}

```

#### 57.至少有 1 位重复的数字（LC）

题目：给定正整数 N，返回小于等于 N 且具有至少 1 位重复数字的正整数。示例 2 输入：100 输出：10 解释：具有至少 1 位重复数字的正数（<= 100）有 11，22，33，44，55，66，77，88，99 和 100

思路：题的思路是将其转换为数位DP，利用排列组合完成解题，反过来找无重复的排列组合

```java
package com.qiming.algorithm.leetcode;

import java.util.ArrayList;
import java.util.List;

/**
 * 至少有 1 位重复的数字
 *
 * 给定正整数 N，返回小于等于 N 且具有至少 1 位重复数字的正整数。
 *
 * 示例 2 输入：100 输出：10 解释：具有至少 1 位重复数字的正数（<= 100）有 11，22，33，44，55，66，77，88，99 和 100
 *
 * 提示： 1 <= N <= 10^9
 *
 * 思路： 本题的思路是将其转换为数位DP，利用排列组合完成解题，题目需要求数字N有多少个重复的数字，可以将其转换为求数字N有多少个不重复的数字，因为求不重复的数字可以更好地使用排列组合来求解
 *
 * 现在我们的重心来到要怎么将这个数字分解成可以按一定规律计算其所有不重复数位的排列组合，总体的思路是：设剩下的位置为i，剩下的数字为j，则不重复数字是在每一位依次填入与前几位不同的数字
 *
 * 即选取剩下的j个数字填入剩下的i个位置，即有A(j, i)种可能，最后将其累加就是不重复数字个数，实际遍历中，我们只需要剩下的位置i这个变量，设数字N的位数为k，则剩下的数字j=10-(k-i)，一共10个数字呗
 *
 * 对于以上思路，我们还可以分为以下两种情况，第一种是高位带0，第二种是高位不带0，我们知道数学中0这个数字比较特别，高位数为0即等于没有高位数，比如0096就是数字96，这个数字尽管两个0重复了
 *
 * 但是这两个0属于高位，所以0096这个数字不是重复数字，即第一种情况允许高位的0可以重复，使用第一种情况求位数小于k的不重复数字的个数：因为最高位总是为0，因此一开始剩下的数字j总是为9个（1-9）
 *
 * 然后剩下的低位可选的数字总共有A(10-1,i)，使用第二种情况求位数为k的不重复数字的个数：一开始剩下的数字j受数字N每位上的数字影响，设N的当前位的数字为n，则j<=n，然后剩下的低位可选的数字总共有A(10-(k-i),i)
 *
 * 第一种情况：
 *
 * 4th 3th 2th 1th total
 *  0   0   0  1-9 9xA(9,0)
 *  0   0  1-9 0-9 9xA(9,1)
 *  0  1-9 0-9 0-9 9xA(9,2)
 *
 * 第二种情况：
 *
 * 4th 3th 2th 1th total
 * 1-2 0-9 0-9 0-9 2xA(9,3)
 *  3  0-4 0-9 0-9 5xA(8,2)
 *  3   5  0-5 0-9 6xA(7,1)
 *  3   5   6  0-1 2xA(6,0)
 *  3   5   6   2  1
 *
 * 注：total为理想的总数，最后还需要将重复的数字剔除，比如第二种情况的第二行中，如果遍历到了33xx，则后面的xx不需要再计算，因为高位的33已经使这个数字变为了重复数字，循环可以直接break掉
 *
 * 比较特殊的情况还有第二种情况的第一行，注意高位是从1开始，因为0的情况在第一种情况的最后一行已经考虑；还有第二种情况的最后一行，如果前三个高位的数字不重复，并且最后要填入的2也与前面数字不重复，
 *
 * 则数字N本身也是一个不重复数字
 *
 */
public class NumbersWithRepeatedDigits {

  public int numDupDigitsAtMostN(int N) {
    return N - dp(N);
  }

  public int dp(int n) {
    List<Integer> digits = new ArrayList<>();
    while (n > 0) {
      digits.add(n % 10);
      n /= 10;
    }
    int k = digits.size();

    int[] used = new int[10];
    int total = 0;

    for (int i = 1; i < k; i++) {
      total += 9 * A(9, i - 1);
    }

    for (int i = k - 1; i >= 0; i--) {
      int num = digits.get(i);

      for (int j = i == k - 1 ? 1 : 0; j < num; j++) {
        if (used[j] != 0) {
          continue;
        }
        total += A(10 - (k - i), i);
      }

      if (++used[num] > 1) {
        break;
      }

      if (i == 0) {
        total += 1;
      }
    }
    return total;
  }

  public int fact(int n) {
    if (n == 1 || n == 0) {
      return 1;
    }
    return n * fact(n - 1);
  }

  public int A(int m, int n) {
    return fact(m) / fact(m - n);
  }


}
```

#### 58.按字典序排在最后的子串（LC）

题目：给你一个字符串 s，找出它的所有子串并按字典序排列，返回排在最后的那个子串。示例 1：输入："abab" 输出："bab" 解释：我们可以找出 7 个子串 ["a", "ab", "aba", "abab", "b", "ba", "bab"]。按字典序排在最后的子串是 "bab"。

思路：要求字典序最大的子串，那么一定是该字符串的一个后缀子串，因为如果是中间的一个字串的话，那么只要向它后面添加字母，它的字典序一定会更大。这个理解，然后问题转化为如何求的字典序最大的后缀子串，就是最大表示法，但是要处理下字符串，最后加一个比'a'小的'.'

```java
package com.qiming.algorithm.leetcode;

/**
 *  按字典序排在最后的子串
 *
 *  给你一个字符串 s，找出它的所有子串并按字典序排列，返回排在最后的那个子串。
 *
 *  示例 1：输入："abab" 输出："bab" 解释：我们可以找出 7 个子串 ["a", "ab", "aba", "abab", "b", "ba", "bab"]。按字典序排在最后的子串是 "bab"。
 *  示例 2: 输入："leetcode" 输出："tcode"
 *
 *  思路：要求字典序最大的子串，那么一定是该字符串的一个后缀子串，因为如果是中间的一个字串的话，那么只要向它后面添加字母，它的字典序一定会更大。这个理解
 *
 *  然后问题转化为如何求的字典序最大的后缀子串，就是最大表示法，但是要处理下字符串，最后加一个比'a'小的'.'
 *
 */
public class LastSubstringInLexicographicalOrder {

  public String lastSubstring(String s) {
    s = s + ".";
    int n = s.length();
    int i = 0, j = 1, k = 0;
    while (i < n && j < n && k < n) {
      int t = s.charAt((i + k) % n) - s.charAt((j + k) % n);
      if (t == 0) {
        k++;
      } else {
        if (t > 0) {
          j += k + 1;
        } else {
          i += k + 1;
        }
        if (i == j) {
          j++;
        }
        k = 0;
      }
    }
    int begin = i < j ? i : j;
    return s.substring(begin);
  }

}

```

#### 59.卡牌分组（LC）

题目：给定一副牌，每张牌上都写着一个整数。此时，你需要选定一个数字 X，使我们可以将整副牌按下述规则分成 1 组或更多组。1、每组都有 X 张牌。 2、组内所有的牌上都写着相同的整数。 仅当你可选的 X >= 2 时返回 true。示例 1：输入：[1,2,3,4,4,3,2,1] 输出：true 解释：可行的分组是 [1,1]，[2,2]，[3,3]，[4,4] 示例 3：输入：[1] 输出：false 解释：没有满足要求的分组。

思路：用最大公约数，还有辅助数组存储

```java
package com.qiming.algorithm.leetcode;

/**
 * 卡牌分组
 *
 * 给定一副牌，每张牌上都写着一个整数。此时，你需要选定一个数字 X，使我们可以将整副牌按下述规则分成 1 组或更多组
 *
 * 1、每组都有 X 张牌。 2、组内所有的牌上都写着相同的整数。 仅当你可选的 X >= 2 时返回 true。
 *
 * 示例 1：输入：[1,2,3,4,4,3,2,1] 输出：true 解释：可行的分组是 [1,1]，[2,2]，[3,3]，[4,4] 示例 3：输入：[1] 输出：false 解释：没有满足要求的分组。
 *
 * 思路：用最大公约数，还有辅助数组存储
 *
 */
public class XOfAKindInADeckOfCards {

  public static void main(String[] args) {
    System.out.println(new XOfAKindInADeckOfCards().gcd(9,6));
  }

  public boolean hasGroupsSizeX(int[] deck) {

    int[] count = new int[10000];

    //用数组空间存储
    for (int i = 0; i < deck.length; i++) {
      count[deck[i]]++;
    }

    int g = -1;
    for (int i = 0; i < 10000; i++) {
      if (count[i] > 0) {
        if (g == -1) {
          g = count[i];
        } else {
          g = gcd(g, count[i]);
        }
      }
    }

    return g >= 2;

  }

  //x,y不分谁大谁小的，求最大公约数
  private int gcd(int x, int y) {
    return x == 0 ? y : gcd(y%x, x);
  }

}

```

#### 60.快乐数（LC）

题目：编写一个算法来判断一个数是不是“快乐数”。一个“快乐数”定义为：对于一个正整数，每一次将该数替换为它每个位置上的数字的平方和，然后重复这个过程直到这个数变为 1，也可能是无限循环但始终变不到 1。如果可以变为 1，那么这个数就是快乐数。

思路：使用快慢指针思想，如果给定的数字最后会一直循环重复，那么快的指针（值）一定会追上慢的指针（值），也就是两者一定会相等。如果没有循环重复，那么最后快慢指针也会相等，且都等于1。

```java
package com.qiming.algorithm.leetcode;

/**
 * 快乐数
 *
 * 编写一个算法来判断一个数是不是“快乐数”。一个“快乐数”定义为：对于一个正整数，每一次将该数替换为它每个位置上的数字的平方和，然后重复这个过程直到这个数变为 1，也可能是无限循环但始终变不到 1。如果可以变为 1，那么这个数就是快乐数。
 *
 * 示例: 输入: 19 输出: true
 *
 * 解释:
 * 12 + 92 = 82
 * 82 + 22 = 68
 * 62 + 82 = 100
 * 12 + 02 + 02 = 1
 *
 * 思路：使用快慢指针思想，如果给定的数字最后会一直循环重复，那么快的指针（值）一定会追上慢的指针（值），也就是两者一定会相等。如果没有循环重复，那么最后快慢指针也会相等，且都等于1。
 *
 */
public class HappyNumber {

  public static void main(String[] args) {
    System.out.println(new HappyNumber().squareSum(19));
  }

  public boolean isHappy(int n) {
    int fast = n;
    int slow = n;
    do {
      slow = squareSum(slow);
      fast = squareSum(fast);
      fast = squareSum(fast);
    } while (slow != fast);
    if (fast == 1) {
      return true;
    } else {
      return false;
    }
  }

  /**
   * 求数字的平方和，跟10有关
   * @param m
   * @return
   */
  private int squareSum(int m) {

    int squareSum = 0;
    while (m != 0) {
      squareSum += (m%10)*(m%10);
      m /= 10;
    }
    return squareSum;
  }

}

```

#### 61.数组拆分 I（LC）

题目：给定长度为 2n 的数组, 你的任务是将这些数分成 n 对, 例如 (a1, b1), (a2, b2), ..., (an, bn) ，使得从1 到 n 的 min(ai, bi) 总和最大。输入: [1,4,3,2] 输出: 4 解释: n 等于 2, 最大总和为 4 = min(1, 2) + min(3, 4).

思路：排序后，每两组较小的数相加即可，用快排

```java
package com.qiming.algorithm.leetcode;

/**
 * 数组拆分 I
 *
 * 给定长度为 2n 的数组, 你的任务是将这些数分成 n 对, 例如 (a1, b1), (a2, b2), ..., (an, bn) ，使得从1 到 n 的 min(ai, bi) 总和最大。
 * 输入: [1,4,3,2] 输出: 4 解释: n 等于 2, 最大总和为 4 = min(1, 2) + min(3, 4).
 *
 * 提示: n 是正整数,范围在 [1, 10000]. 数组中的元素范围在 [-10000, 10000].
 *
 * 思路，排序后，每两组较小的数相加即可，用快排
 *
 */
public class ArrayPartitionI {

  public static void main(String[] args) {
//    int[] a = (4,1,3,6,7,2,9);
    int[] a = (1,2,3,2);
    new ArrayPartitionI().quicksort(a, 0, a.length - 1);
    for (int i = 0; i < a.length; i++) {
      System.out.println(a[i]);
    }
  }

  public int arrayPairSum(int[] nums) {
    quicksort(nums, 0, nums.length - 1);
    int sum = 0;
    for (int i = 0; i < nums.length; i = i + 2) {
      sum += nums[i];
    }
    return sum;
  }

  private void quicksort(int[] sums, int low, int high) {
    if (low < high) {
      int partition = partition(sums, low, high);
      quicksort(sums, low, partition - 1);
      quicksort(sums, partition + 1, high);
    }
  }

  private int partition(int[] array, int low, int high) {
    int poivt = array[low];
    while (low < high) {
      while (low < high && array[high] >= poivt) {
        high--;
      }
      array[low] = array[high];
      while (low < high && array[low] <= poivt) {
        low++;
      }
      array[high] = array[low];
    }
    array[low] = poivt;
    return low;
  }

}

```

#### 62.区间子数组个数（LC）

题目：给定一个元素都是正整数的数组A ，正整数 L 以及 R (L <= R)。求连续、非空且其中最大元素满足大于等于L 小于等于R的子数组个数。

思路：其实可以用标记，小于L的标0，L和R之间的标1，大于R的标2，就是求被2分割的，子数组里至少包含一个1的个数，然后就转换解题思路

```java
package com.qiming.algorithm.leetcode;

/**
 * 区间子数组个数
 *
 * 给定一个元素都是正整数的数组A ，正整数 L 以及 R (L <= R)。求连续、非空且其中最大元素满足大于等于L 小于等于R的子数组个数。
 * 输入:
 * A = [2, 1, 4, 3]
 * L = 2
 * R = 3
 * 输出: 3
 * 解释: 满足条件的子数组: [2], [2, 1], [3].
 *
 * L, R  和 A[i] 都是整数，范围在 [0, 10^9]。 数组 A 的长度范围在[1, 50000]。
 *
 * 思路 其实可以用标记，小于L的标0，L和R之间的标1，大于R的标2，就是求被2分割的，子数组里至少包含一个1的个数，然后就转换成
 *
 * 最大元素满足大于等于L小于等于R的子数组个数 = 最大元素小于等于R的子数组个数 - 最大元素小于L的子数组个数（因为最大元素小于L的子数组个数肯定包含在最大元素小于等于R的子数组个数）
 *
 */
public class NumberOfSubarraysWithBoundedMaximum {

  public int numSubarrayBoundedMax(int[] A, int L, int R) {

    return numSubarrayBoundedMax(A, R) - numSubarrayBoundedMax(A, L-1);

  }

  private int numSubarrayBoundedMax(int[] A, int Max) {
    int res = 0;
    int numSubarray = 0;
    for (int i = 0; i < A.length; i++) {
      if (A[i] <= Max) {
        numSubarray++;
        res += numSubarray;
      } else {
        numSubarray = 0;
      }
    }
    return res;
  }

  /**
   * 暴力解法，肯定超出时间
   * @param A
   * @param L
   * @param R
   * @return
   */
  public int numSubarrayBoundedMaxByViolence(int[] A, int L, int R) {

    int result = 0;

    for (int i = 0; i < A.length; i++) {
      int max = A[i];
      for (int j = i; j < A.length; j++) {
        max = Math.max(A[j], max);
        if (L <= max && max <= R) {
          result++;
        }
      }
    }

    return result;

  }

}

```

#### 63.判断一棵树是否是完全二叉树

题目：判断一棵树是否是完全二叉树

思路：有四种情况，左空右空，左空右有，左有右空，左有右有。左空右有肯定不是完全二叉树了，左有右空和左空右空说明下一个节点(层次)肯定是叶子结点。此时看队列里剩余的元素，应该都是叶子了，如有不是，就不是完全

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 判断一颗二叉树是不是完全二叉树
 *
 *
 * 思路：有四种情况，左空右空，左空右有，左有右空，左有右有
 * 左空右有肯定不是完全二叉树了，左有右空和左空右空说明下一个节点(层次)肯定是叶子结点。此时看队列里剩余的元素，应该都是叶子了，如有不是，就不是完全
 *
 *
 */
public class IsCompleteBinTree {

  public static void main(String[] args) {

  }

  private boolean isCompleteBinTree(TreeNodeIsCompleteBinTree root) {
    if (root == null) {
      return true;
    }
    Queue<TreeNodeIsCompleteBinTree> queue = new LinkedList();
    queue.offer(root);
    while (!queue.isEmpty()) {
      TreeNodeIsCompleteBinTree p = queue.poll();
      if (p.left != null) {
        queue.offer(p.left);
        if (p.right != null) {
          //左有右有
          queue.offer(p.right);
        } else {
          //左有右空
          break;
        }
      } else {
        if (p.right != null) {
          //左无右有
          return false;
        } else {
          //左无右无
          break;
        }
      }
    }

    while (!queue.isEmpty()) {
      TreeNodeIsCompleteBinTree p = queue.poll();
      if (p.left != null || p.right != null) {
        return false;
      }
    }
    //单结点，为完全二叉树
    return true;
  }

}

class TreeNodeIsCompleteBinTree{

  int val;
  TreeNodeIsCompleteBinTree left;
  TreeNodeIsCompleteBinTree right;

  public TreeNodeIsCompleteBinTree(int val) {
    this.val = val;
  }
}

```

#### 64.验证外星语词典（LC）

题目：某种外星语也使用英文小写字母，但可能顺序 order 不同。字母表的顺序（order）是一些小写字母的排列。给定一组用外星语书写的单词 words，以及其字母表的顺序 order，只有当给定的单词在这种外星语中按字典序排列时，返回 true；否则，返回 false。

思路：一开始同hashmap做词典，后来改成数组了

```java
package com.qiming.algorithm.leetcode;

import java.util.HashMap;
import java.util.Map;

/**
 * 验证外星语词典
 *
 * 某种外星语也使用英文小写字母，但可能顺序 order 不同。字母表的顺序（order）是一些小写字母的排列。
 * 给定一组用外星语书写的单词 words，以及其字母表的顺序 order，只有当给定的单词在这种外星语中按字典序排列时，返回 true；否则，返回 false。
 *
 * 输入：words = ["hello","leetcode"], order = "hlabcdefgijkmnopqrstuvwxyz" 输出：true 解释：在该语言的字母表中，'h' 位于 'l' 之前，所以单词序列是按字典序排列的
 * 输入：words = ["word","world","row"], order = "worldabcefghijkmnpqstuvxyz" 输出：false 解释：在该语言的字母表中，'d' 位于 'l' 之后，那么 words[0] > words[1]，因此单词序列不是按字典序排列的。
 * 输入：words = ["apple","app"], order = "abcdefghijklmnopqrstuvwxyz" 输出：false 解释：当前三个字符 "app" 匹配时，第二个字符串相对短一些，然后根据词典编纂规则 "apple" > "app"，因为 'l' > '∅'，其中 '∅' 是空白字符，定义为比任何其他字符都小（更多信息）。
 *
 * 1 <= words.length <= 100   1 <= words[i].length <= 20   order.length == 26   在 words[i] 和 order 中的所有字符都是英文小写字母。
 *
 * 思路：一开始同hashmap做词典，后来改成数组了
 *
 */
public class VerifyingAnAlienDictionary {

  public static void main(String[] args) {
//    String[] s = {"apple", "app"};
//    String order = "abcdefghijklmnopqrstuvwxyz";

    String[] s = {"hello","leetcode"};
    String order = "hlabcdefgijkmnopqrstuvwxyz";

    new VerifyingAnAlienDictionary().isAlienSorted(s, order);
  }

  public boolean isAlienSorted(String[] words, String order) {

    /**
     * 这个太耗时间，HashMap也不可取，还得用数组
     */
//    if (words.length == 1) {
//      return true;
//    }
//
//    Map<Character, Integer> index = new HashMap();
//    for (int i = 0; i < order.length(); i++) {
//      index.put(order.charAt(i), i);
//    }
//
//    for (int i = 0, j = 1; i < words.length - 1; i++, j++) {
//      if (compareTwoString(words[i], words[j], index) == 1) {
//        return false;
//      }
//    }
//
//    return true;

    if (words.length == 1) {
      return true;
    }

    //上数组
    int[] index = new int[26];
    for (int i = 0; i < order.length(); i++) {
      //这样就变成索引了
      index[order.charAt(i) - 'a'] = i; //i在index数组代表原a-z现在是第几位
    }


    for (int i = 0, j = 1; i < words.length - 1; i++, j++) {

      String s1 = words[i];
      String s2 = words[j];

      boolean breakFlag = false;
      for (int k = 0; k < Math.min(s1.length(), s2.length()); k++) {
        if (s1.charAt(k) != s2.charAt(k)) {
          if (index[s1.charAt(k) - 'a'] > index[s2.charAt(k) - 'a']) {
            return false;
          } else {
            //这个地方两种情况整不明白，就用一个flag
            breakFlag = true;
            break;
          }
        }
      }

      if (breakFlag) {
        continue;
      }
      if (s1.length() > s2.length()) {
        return false;
      }
    }

    return true;

  }

  private int compareTwoString(String s1, String s2, Map<Character, Integer> index) {
    int s1len = s1.length(), s2len = s2.length();
    int len = s1len >= s2len ? s2len : s1len;
    for (int i = 0; i < len; i++) {
      int tmp = index.get(s1.charAt(i)) - index.get(s2.charAt(i));
      if (tmp > 0) {
        return 1;
      } else if (tmp < 0) {
        return -1;
      }
    }
    if (s1len == s2len) {
      return 0;
    }

    return s1len > s2len ? 1 : -1;
  }


}

```

#### 65.二叉搜索树结点最小距离（LC）

题目：给定一个二叉搜索树的根结点 root, 返回树中任意两节点的差的最小值。输入: root = [4,2,6,1,3,null,null] 输出: 1  解释: 注意，root是树结点对象(TreeNode object)，而不是数组。

思路：首先看清是二叉搜索树，然后就是中序遍历从小到大，最小就是相邻两个差值的最小

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.Queue;
import java.util.Stack;

/**
 * 二叉搜索树结点最小距离
 *
 * 给定一个二叉搜索树的根结点 root, 返回树中任意两节点的差的最小值。
 *
 * 输入: root = [4,2,6,1,3,null,null] 输出: 1  解释: 注意，root是树结点对象(TreeNode object)，而不是数组。
 *
 * 二叉树的大小范围在 2 到 100。 二叉树总是有效的，每个节点的值都是整数，且不重复。
 *
 *
 * 思路：首先看清是二叉搜索树，然后就是中序遍历从小到大，最小就是相邻两个差值的最小
 *
 */
public class MinimumDistanceBetweenBSTNodes {

  public static void main(String[] args) {
    TreeNodeMinimumDistanceBetweenBSTNodes t1 = new TreeNodeMinimumDistanceBetweenBSTNodes(90);
    TreeNodeMinimumDistanceBetweenBSTNodes t2 = new TreeNodeMinimumDistanceBetweenBSTNodes(69);
    TreeNodeMinimumDistanceBetweenBSTNodes t3 = new TreeNodeMinimumDistanceBetweenBSTNodes(49);
    TreeNodeMinimumDistanceBetweenBSTNodes t4 = new TreeNodeMinimumDistanceBetweenBSTNodes(89);
    TreeNodeMinimumDistanceBetweenBSTNodes t5 = new TreeNodeMinimumDistanceBetweenBSTNodes(52);

    t1.left = t2;
    t2.left = t3;
    t2.right = t4;
    t3.right = t5;

    new MinimumDistanceBetweenBSTNodes().minDiffInBST(t1);

  }


  public int minDiffInBST(TreeNodeMinimumDistanceBetweenBSTNodes root) {

    //二叉树的中序遍历
    Stack<TreeNodeMinimumDistanceBetweenBSTNodes> stack = new Stack();
    TreeNodeMinimumDistanceBetweenBSTNodes p = root;
    Integer minMax = Integer.MAX_VALUE;
    Integer pre = null;
    while (p != null || !stack.isEmpty()) {
      //左结点全部入栈
      while (p != null) {
        stack.push(p);
        p = p.left;
      }
      if (!stack.isEmpty()) {
        //这边也要是p哈
        p = stack.pop();
        //增加，在这边有个小技巧，就是我不想第一个值减初始的pre，而是在上面把pre置null，然后判断一下
        if (pre != null) {
          minMax = Math.min(p.val - pre, minMax);
        }
        pre = p.val;

        p = p.right;
      }
    }
    return minMax;
  }

}

class TreeNodeMinimumDistanceBetweenBSTNodes {

  int val;
  TreeNodeMinimumDistanceBetweenBSTNodes left;
  TreeNodeMinimumDistanceBetweenBSTNodes right;
  TreeNodeMinimumDistanceBetweenBSTNodes(int x) { val = x; }

}

```
#### 66.计算各个位数不同的数字个数（LC）

题目：给定一个非负整数 n，计算各位数字都不同的数字 x 的个数，其中 0 ≤ x < 10^n 。示例: 输入: 2 输出: 91 解释: 答案应为除去 11,22,33,44,55,66,77,88,99 外，在 [0,100) 区间内的所有数字。

思路：数学题，看下方详情

```java
package com.qiming.algorithm.leetcode;

/**
 * 计算各个位数不同的数字个数
 *
 * 给定一个非负整数 n，计算各位数字都不同的数字 x 的个数，其中 0 ≤ x < 10^n 。
 *
 * 示例: 输入: 2 输出: 91 解释: 答案应为除去 11,22,33,44,55,66,77,88,99 外，在 [0,100) 区间内的所有数字。
 *
 * 思路：数学题
 *
 * n=1: res=10
 *
 * n=2: res=9*9+10=91 # 两位数第一位只能为1-9，第二位只能为非第一位的数，加上一位数的所有结果
 *
 * n=3: res=9 * 9 * 8+91=739 # 三位数第一位只能为1-9，第二位只能为非第一位的数，第三位只能为非前两位的数，加上两位数的所有结果
 *
 * n=4: res=9 * 9 * 8 * 7+739=5275 # 同上推法
 *
 * n>10: 就不能有都不一样的数字了，所以有Math.min()
 *
 */
public class CountNumbersWithUniqueDigits {

  public int countNumbersWithUniqueDigits(int n) {
    if (n == 0) {
      return 1;
    }
    int mul = 9;
    int result = 10;
    for (int i = 1; i < Math.min(n, 10); i++) {
      mul *= 10 - i;
      result += mul;
    }
    return result;
  }

}

```

#### 67.子数组最大平均数 I（LC）

题目：给定 n 个整数，找出平均数最大且长度为 k 的连续子数组，并输出该最大平均数。示例 1: 输入: [1,12,-5,-6,50,3], k = 4  输出: 12.75  解释: 最大平均数 (12-5-6+50)/4 = 51/4 = 12.75

思路：自己的思路不好，一个是空间换时间一个是滑动窗口

```java
package com.qiming.algorithm.leetcode;

/**
 * 子数组最大平均数 I
 *
 * 给定 n 个整数，找出平均数最大且长度为 k 的连续子数组，并输出该最大平均数。
 *
 * 示例 1: 输入: [1,12,-5,-6,50,3], k = 4  输出: 12.75  解释: 最大平均数 (12-5-6+50)/4 = 51/4 = 12.75
 *
 * 注意:  1 <= k <= n <= 30,000。  所给数据范围 [-10,000，10,000]。
 *
 * 思路：自己的思路不好，一个是空间换时间一个是滑动窗口
 *
 */
public class MaximumAverageSubarrayI {

  public static void main(String[] args) {
    int nums[] = {-1};
    int k = 1;
    System.out.println(new MaximumAverageSubarrayI().findMaxAverage(nums, k));
  }

  public double findMaxAverage(int[] nums, int k) {

    //看题目要求，但是这样写不好
//    double maxAvg = -10000;
//    for (int i = 0; i < nums.length - k + 1; i++) {
//      double tmp = 0;
//      for (int j = i; j < i + k; j++) {
//        tmp += nums[j];
//      }
//      tmp /= k;
//      maxAvg = Math.max(maxAvg, tmp);
//    }
//    return maxAvg;

    //比较好的想法1，辅助数组，空间换时间，记录从0到i数的和先sum，那么k个数字的总和就是sum[i] - sum[i-k]
//    int[] sum = new int[nums.length];
//    sum[0] = nums[0];
//    for (int i = 1; i < nums.length; i++) {
//      sum[i] = sum[i - 1] + nums[i];
//    }
//    double res = sum[k - 1] * 1.0 / k;
//    for (int i = k; i < nums.length; i++) {
//      res = Math.max(res, (sum[i] - sum[i - k]) * 1.0 / k);
//    }
//    return res;

    //比较好的想法2，滑动窗口，这个可以想，如果i到i+k的子数组和为x，那么i+1到i+k+1的子数组和就是从x减去i和加上i+k+1
    double sum = 0;
    //第一个累加和
    for (int i = 0; i < k; i++) {
      sum += nums[i];
    }
    double res = sum;
    for (int i = k; i < nums.length; i++) {
      sum += nums[i] - nums[i-k];
      res = Math.max(res, sum);
    }
    return res/k;
  }

}

```
#### 68.删列造序（LC）

题目：给定由 N 个小写字母字符串组成的数组 A，其中每个字符串长度相等。删除 操作的定义是：选出一组要删掉的列，删去 A 中对应列中的所有字符

思路：正常思路，暴力去保持每一列都符合，不符合就++

```java
package com.qiming.algorithm.leetcode;

/**
 * 删列造序
 *
 * 给定由 N 个小写字母字符串组成的数组 A，其中每个字符串长度相等。删除 操作的定义是：选出一组要删掉的列，删去 A 中对应列中的所有字符
 *
 * 形式上，第 n 列为 [A[0][n], A[1][n], ..., A[A.length-1][n]]）。比如，有 A = ["abcdef", "uvwxyz"]， 要删掉的列为 {0, 2, 3}，删除后 A 为["bef", "vyz"]， A 的列分别为["b","v"], ["e","y"], ["f","z"]。
 *
 * 你需要选出一组要删掉的列 D，对 A 执行删除操作，使 A 中剩余的每一列都是 非降序 排列的，然后请你返回 D.length 的最小可能值。
 *
 *
 * 思路，正常思路，暴力去保持每一列都符合，不符合就++
 *
 */
public class DeleteColumnsToMakeSorted {

  public int minDeletionSize(String[] A) {

    int columnLen = A.length;
    int wordLen = A[0].length();
    int deleted = 0;
    for (int i = 0; i < wordLen; i++) {
      Character pre = null;
      for (int j = 0; j < columnLen; j++) {
        if (pre != null) {
          if (A[j].charAt(i) - A[j-1].charAt(i) < 0) {
            deleted++;
            break;
          }
        }
        pre = A[j].charAt(i);
      }
    }
    return deleted;
  }

}

```

#### 69.字符串中的单词数（LC）

题目：统计字符串中的单词个数，这里的单词指的是连续的不是空格的字符。请注意，你可以假定字符串里不包括任何不可打印的字符。示例: 输入: "Hello, my name is John" 输出: 5

思路：找单词的起始标志，是下标前为空格，下标不为空格，就是字符的开始，但是当0下标是空格的时候也要特殊判断一下

```java
package com.qiming.algorithm.leetcode;

/**
 * 字符串中的单词数
 *
 * 统计字符串中的单词个数，这里的单词指的是连续的不是空格的字符。请注意，你可以假定字符串里不包括任何不可打印的字符。
 *
 * 示例: 输入: "Hello, my name is John" 输出: 5
 *
 * 思路：找单词的起始标志，是下标前为空格，下标不为空格，就是字符的开始，但是当0下标是空格的时候也要特殊判断一下
 *
 *
 */
public class NumberOfSegmentsInAString {

  public static void main(String[] args) {
//    String s = "";
    String s = ", , , ,        a, eaefa";
    System.out.println(new NumberOfSegmentsInAString().countSegments(s));
  }

  public int countSegments(String s) {
    int segmentCount = 0;

    for (int i = 0; i < s.length(); i++) {
      if ((i == 0 || s.charAt(i-1) == ' ') && s.charAt(i) != ' ') {
        segmentCount++;
      }
    }

    return segmentCount;
  }

}

```

#### 70.最长连续递增序列（LC）

题目：给定一个未经排序的整数数组，找到最长且连续的的递增序列。输入: [1,3,5,4,7] 输出: 3   输入: [2,2,2,2,2]  输出: 1

思路：类似滑动窗口一类的解题思路

```java
package com.qiming.algorithm.leetcode;

/**
 * 最长连续递增序列
 *
 * 给定一个未经排序的整数数组，找到最长且连续的的递增序列。
 *
 * 输入: [1,3,5,4,7] 输出: 3   输入: [2,2,2,2,2]  输出: 1
 *
 * 思路：类似滑动窗口一类的解题思路
 *
 */
public class LongestContinuousIncreasingSubsequence {

  public static void main(String[] args) {
    int nums[] = {1,3,5,4,2,3,4,5};
    System.out.println(new LongestContinuousIncreasingSubsequence().findLengthOfLCIS(nums));
  }

  public int findLengthOfLCIS(int[] nums) {

    /**
     * 自己写的
     */
//    if (nums.length == 0) {
//      return 0;
//    }
//
//    if (nums.length == 1) {
//      return 1;
//    }
//
//    int longest = 1;
//    int tmp = 1;
//    for (int i = 1; i < nums.length; i++) {
//      if ((nums[i] - nums[i-1]) > 0) {
//        tmp++;
//      } else {
//        longest = Math.max(longest, tmp);
//        //注意这边tmp为1
//        tmp = 1;
//      }
//    }
//    //最后一定要记得比较下，因为有一直增的情况
//    longest = Math.max(longest, tmp);
//    return longest;

    /**
     * 标准滑动窗口解答，上面的tmp用下标做替换
     */

    int ans = 0, anchor = 0;
    for (int i = 0; i < nums.length; ++i) {
      //这边if i > 0的技巧，跳过了没有值和一个值
      if (i > 0 && nums[i-1] >= nums[i]) anchor = i;
      ans = Math.max(ans, i - anchor + 1);
    }
    return ans;
  }

}

#### 71.奇数值单元格的数目（LC）

题目：给你一个 n 行 m 列的矩阵，最开始的时候，每个单元格中的值都是 0。另有一个索引数组 indices，indices[i] = [ri, ci] 中的 ri 和 ci 分别表示指定的行和列（从 0 开始编号）。你需要将每对 [ri, ci] 指定的行和列上的所有单元格的值加 1。请你在执行完所有 indices 指定的增量操作后，返回矩阵中 「奇数值单元格」 的数目。

思路：画几个图就知道了，顺序不重要，最后是奇数就行了，挖掉横竖相交的部分，不要忘了乘以2，重合了两次

```java
package com.qiming.algorithm.leetcode;

/**
 * 奇数值单元格的数目
 *
 * 给你一个 n 行 m 列的矩阵，最开始的时候，每个单元格中的值都是 0。另有一个索引数组 indices，indices[i] = [ri, ci] 中的 ri 和 ci 分别表示指定的行和列（从 0 开始编号）。
 *
 * 你需要将每对 [ri, ci] 指定的行和列上的所有单元格的值加 1。请你在执行完所有 indices 指定的增量操作后，返回矩阵中 「奇数值单元格」 的数目。
 *
 * 提示：1 <= n <= 50 1 <= m <= 50 1 <= indices.length <= 100  0 <= indices[i][0] < n  0 <= indices[i][1] < m
 *
 * 思路：画几个图就知道了，顺序不重要，最后是奇数就行了，挖掉横竖相交的部分，不要忘了乘以2，重合了两次
 *
 */
public class CellsWithOddValuesInAMatrix {

  public static void main(String[] args) {
    int n = 2;
    int m = 2;
    int[][] indices = ({1,1},{0,0}); //jekyll
    System.out.println(new CellsWithOddValuesInAMatrix().oddCells(n,m,indices));
  }

  public int oddCells(int n, int m, int[][] indices) {

    int row[] = new int[50];
    int column[] = new int[50];

    for (int i = 0; i < indices.length; i++) {
      row[indices[i][0]]++;
      column[indices[i][1]]++;
    }
    int rowOdd = 0;
    int columnOdd = 0;
    for (int i = 0; i < row.length; i++) {
      if (row[i] % 2 == 1) {
        rowOdd++;
      }
      if (column[i] % 2 == 1) {
        columnOdd++;
      }
    }

    return rowOdd * m + columnOdd * n - rowOdd * columnOdd *2;

  }

}

```
#### 72.判断子序列（LC）

题目：给定字符串 s 和 t ，判断 s 是否为 t 的子序列。你可以认为 s 和 t 中仅包含英文小写字母。字符串 t 可能会很长（长度 ~= 500,000），而 s 是个短字符串（长度 <=100）。字符串的一个子序列是原始字符串删除一些（也可以不删除）字符而不改变剩余字符相对位置形成的新字符串。（例如，"ace"是"abcde"的一个子序列，而"aec"不是）。

思路：两个指针，如果s的指针走完了，说明在t中能找到这样的序列

```java
package com.qiming.algorithm.leetcode;

/**
 * 判断子序列
 *
 * 给定字符串 s 和 t ，判断 s 是否为 t 的子序列。你可以认为 s 和 t 中仅包含英文小写字母。字符串 t 可能会很长（长度 ~= 500,000），而 s 是个短字符串（长度 <=100）。
 *
 * 字符串的一个子序列是原始字符串删除一些（也可以不删除）字符而不改变剩余字符相对位置形成的新字符串。（例如，"ace"是"abcde"的一个子序列，而"aec"不是）。
 *
 * 示例 1:  s = "abc", t = "ahbgdc"  返回 true.
 *
 * 思路：两个指针，如果s的指针走完了，说明在t中能找到这样的序列
 *
 */
public class IsSubsequence {

  public boolean isSubsequence(String s, String t) {
    int i = 0,j = 0;
    while (i < s.length() && j < t.length()) {
      if (s.charAt(i) == t.charAt(j)) {
        i++;
      }
      j++;
    }
    if (i == s.length()) {
      return true;
    } else {
      return false;
    }
  }

}

```

#### 73.区域和检索 - 数组不可变（LC）

题目：给定一个整数数组  nums，求出数组从索引 i 到 j  (i ≤ j) 范围内元素的总和，包含 i,  j 两点。示例： 给定 nums = [-2, 0, 3, -5, 2, -1]，求和函数为 sumRange()  sumRange(0, 2) -> 1  sumRange(2, 5) -> -1  sumRange(0, 5) -> -3，你可以假设数组不可变。会多次调用 sumRange 方法。

思路：暴力办法是超时的，i-j的和相当于sum[j] - sum[i-1]，sum为0到下标的和

```java
package com.qiming.algorithm.leetcode;

/**
 * 区域和检索 - 数组不可变
 *
 * 给定一个整数数组  nums，求出数组从索引 i 到 j  (i ≤ j) 范围内元素的总和，包含 i,  j 两点。
 *
 * 示例： 给定 nums = [-2, 0, 3, -5, 2, -1]，求和函数为 sumRange()  sumRange(0, 2) -> 1  sumRange(2, 5) -> -1  sumRange(0, 5) -> -3
 *
 * 你可以假设数组不可变。会多次调用 sumRange 方法。
 *
 * 思路：暴力办法是超时的，i-j的和相当于sum[j] - sum[i-1]，sum为0到下标的和
 *
 */
public class RangeSumQueryImmutable {

  private static int[] sums;

  public RangeSumQueryImmutable(int[] nums) {
//    this.nums = nums;
    sums = new int[nums.length + 1];
    for (int i = 0; i < nums.length; i++) {
      sums[i+1] = sums[i] + nums[i];
    }
  }

  public int sumRange(int i, int j) {
//    int sum = 0;
//    for (int k = i; k <= j ; k++) {
//      sum += nums[k];
//    }
//    return sum;
    return sums[j+1] - sums[i];
  }

}

/**
 * Your NumArray object will be instantiated and called as such:
 * NumArray obj = new NumArray(nums);
 * int param_1 = obj.sumRange(i,j);
 */

```

#### 74.不用加减乘除做加法（LC）

题目：写一个函数，求两个整数之和，要求在函数体内不得使用 “+”、“-”、“*”、“/” 四则运算符号。

思路：要知道这个a + b等价于计算(a ^ b) + ((a & b) << 1)，其中((a & b) << 1)表示进位。因此令a等于(a & b) << 1，令b等于a ^ b，直到a变成0，然后返回b。

```java
package com.qiming.algorithm.leetcode;

/**
 * 不用加减乘除做加法
 *
 * 写一个函数，求两个整数之和，要求在函数体内不得使用 “+”、“-”、“*”、“/” 四则运算符号。
 *
 * 示例: 输入: a = 1, b = 1 输出: 2
 *
 * 提示：a, b 均可能是负数或 0  结果不会溢出 32 位整数
 *
 * 思路：要知道这个a + b等价于计算(a ^ b) + ((a & b) << 1)，其中((a & b) << 1)表示进位。因此令a等于(a & b) << 1，令b等于a ^ b，直到a变成0，然后返回b。
 *
 */
public class AddWithNoNormal {

  public int add(int a, int b) {
    while (a != 0) {
      int temp = a ^ b;
      a = (a & b) << 1;
      b = temp;
    }
    return b;
  }

}

```

#### 75.递增的三元子序列（LC）

题目：给定一个未排序的数组，判断这个数组中是否存在长度为 3 的递增子序列。数学表达式如下: 如果存在这样的 i, j, k,  且满足 0 ≤ i < j < k ≤ n-1，使得 arr[i] < arr[j] < arr[k] ，返回 true ; 否则返回 false 。说明: 要求算法的时间复杂度为 O(n)，空间复杂度为 O(1) 。

思路：注意下标是可以不连续的，只要有子序列就行的，变换small和big，一直保持一对递增的，如果小于这对递增的最小值，就更新最小值，如果介于最小值和递增值之间(中间值)，就更新递增值(中间值)。最后只要有一个大于中间值的就行了

```java
package com.qiming.algorithm.leetcode;

/**
 * 递增的三元子序列
 *
 * 给定一个未排序的数组，判断这个数组中是否存在长度为 3 的递增子序列。
 *
 * 数学表达式如下: 如果存在这样的 i, j, k,  且满足 0 ≤ i < j < k ≤ n-1，使得 arr[i] < arr[j] < arr[k] ，返回 true ; 否则返回 false 。说明: 要求算法的时间复杂度为 O(n)，空间复杂度为 O(1) 。
 *
 * 示例 1: 输入: [1,2,3,4,5] 输出: true
 *
 * 思路，注意下标是可以不连续的，只要有子序列就行的，变换small和big，一直保持一对递增的，如果小于这对递增的最小值，就更新最小值，如果介于最小值和递增值之间(中间值)，就更新递增值(中间值)
 *
 * 最后只要有一个大于中间值的就行了
 *
 */
public class IncreasingTripletSubsequence {

  public boolean increasingTriplet(int[] nums) {
    int i = 0;
    int small = Integer.MAX_VALUE, big = Integer.MAX_VALUE;

    while (i < nums.length) {
      if (nums[i] < small) {
        small = nums[i];
      } else if (nums[i] > small && nums[i] <= big) {
        big = nums[i];
      } else if (nums[i] > big) {
        return true;
      }
      i++;
    }
    return false;
  }

}

```

#### 76.打印零与奇偶数（LC）

题目：相同的一个 ZeroEvenOdd 类实例将会传递给三个不同的线程：每个线程都有一个 printNumber 方法来输出一个整数。请修改给出的代码以输出整数序列 010203040506... ，其中序列的长度必须为 2n。

思路：lock的condition正常写，但是会有超时，不加锁去写，有volatile写法和Semaphore(模拟锁)写法

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

#### 77.两数之和 II - 输入有序数组（LC）

题目：给定一个已按照升序排列 的有序数组，找到两个数使得它们相加之和等于目标数。函数应该返回这两个下标值 index1 和 index2，其中 index1 必须小于 index2。返回的下标值（index1 和 index2）不是从零开始的。你可以假设每个输入只对应唯一的答案，而且你不可以重复使用相同的元素。

思路：自己写的速度不行，那么还是要想到双指针法，这边的思想是一个是小指针，一个是大指针，如果和大于target，则大指针就减小，如果小于target，就小指针增大

```java
package com.qiming.algorithm.leetcode;

/**
 * 两数之和 II - 输入有序数组
 *
 * 给定一个已按照升序排列 的有序数组，找到两个数使得它们相加之和等于目标数。函数应该返回这两个下标值 index1 和 index2，其中 index1 必须小于 index2。
 *
 * 返回的下标值（index1 和 index2）不是从零开始的。你可以假设每个输入只对应唯一的答案，而且你不可以重复使用相同的元素。
 *
 * 思路：自己写的速度不行，那么还是要想到双指针法，这边的思想是一个是小指针，一个是大指针，如果和大于target，则大指针就减小，如果小于target，就小指针增大
 *
 */
public class TwoSumIIInputArrayIsSorted {

  public int[] twoSum(int[] numbers, int target) {

    /**
     * 自己的简单写法，不算最优
     */
//    if (numbers.length < 2) {
//      return new int[2];
//    }
//
//    int[] result = new int[2];
//    for (int i = 0; i < numbers.length - 1; i++) {
//      for (int j = i + 1; j < numbers.length; j++) {
//        if (numbers[i] + numbers[j] == target) {
//          result[0] = i + 1;
//          result[1] = j + 1;
//        } else if (numbers[i] + numbers[j] > target) {
//          break;
//        }
//      }
//    }
//
//    return result;

    /**
     * 双指针法
     */
    int[] result = new int[2];
    int low = 0, high = numbers.length - 1;
    while (low < high) {
      int sum = numbers[low] + numbers[high];
      if (sum == target) {
        result[0] = low + 1;
        result[1] = high + 1;
        return result;
      } else if (sum < target) {
        low++;
      } else {
        high--;
      }
    }
    return result;
  }

}

```

#### 78.替换空格（LC）

题目：请实现一个函数，把字符串 s 中的每个空格替换成"%20"。输入：s = "We are happy."  输出："We%20are%20happy."

思路：用一个sb即可，正则，replace应该都行

```java
package com.qiming.algorithm.leetcode;

/**
 * 替换空格
 *
 * 请实现一个函数，把字符串 s 中的每个空格替换成"%20"。
 *
 * 输入：s = "We are happy."  输出："We%20are%20happy."
 *
 *
 * 思路：用一个sb即可，正则，replace应该都行
 *
 */
public class ReplaceSpace {

  public String replaceSpace(String s) {

    StringBuilder sb = new StringBuilder();

    for (int i = 0; i < s.length(); i++) {
      if (s.charAt(i) == ' ') {
        sb.append("%20");
      } else {
        sb.append(s.charAt(i));
      }
    }

    return sb.toString();


  }

}

```

#### 79.一手顺子（LC）

题目：爱丽丝有一手（hand）由整数数组给定的牌。 现在她想把牌重新排列成组，使得每个组的大小都是 W，且由 W 张连续的牌组成。如果她可以完成分组就返回 true，否则返回 false。输入：hand = [1,2,3,6,2,3,4,7,8], W = 3   输出：true  解释：爱丽丝的手牌可以被重新排列为 [1,2,3]，[2,3,4]，[6,7,8]。

思路：TreeMap的一个应用，因为手中最小的牌也一定是某个分组中的起始牌，所以反复从手中最小的牌开始组建一个长度为 W 的组。建不出来就返回false

```java
package com.qiming.algorithm.leetcode;

import java.util.TreeMap;

/**
 * 一手顺子
 *
 * 爱丽丝有一手（hand）由整数数组给定的牌。 现在她想把牌重新排列成组，使得每个组的大小都是 W，且由 W 张连续的牌组成。
 *
 * 如果她可以完成分组就返回 true，否则返回 false。
 *
 * 输入：hand = [1,2,3,6,2,3,4,7,8], W = 3   输出：true  解释：爱丽丝的手牌可以被重新排列为 [1,2,3]，[2,3,4]，[6,7,8]。
 *
 * 思路：TreeMap的一个应用，因为手中最小的牌也一定是某个分组中的起始牌，所以反复从手中最小的牌开始组建一个长度为 W 的组。建不出来就返回false
 *
 *
 */
public class HandOfStraights {

  public boolean isNStraightHand(int[] hand, int W) {

    TreeMap<Integer, Integer> count = new TreeMap();
    //现将所有的数据和个数的映射存到treemap中，有序的
    for (int card : hand) {
      if (!count.containsKey(card)) {
        count.put(card, 1);
      } else {
        count.replace(card, count.get(card) + 1);
      }
    }

    while (count.size() > 0) {
      int first = count.firstKey();
      for (int card = first; card < first + W; card++) {
        if (!count.containsKey(card)) {
          return false;
        }
        int c = count.get(card);
        if (c == 1) {
          count.remove(card);
        } else {
          count.replace(card, c - 1);
        }
      }
    }

    return true;

  }

}

```

#### 80.提莫攻击（LC）

题目：在《英雄联盟》的世界中，有一个叫 “提莫” 的英雄，他的攻击可以让敌方英雄艾希（编者注：寒冰射手）进入中毒状态。现在，给出提莫对艾希的攻击时间序列和提莫攻击的中毒持续时间，你需要输出艾希的中毒状态总时长。你可以认为提莫在给定的时间点进行攻击，并立即使艾希处于中毒状态。输入: [1,4], 2  输出: 4  原因: 在第 1 秒开始时，提莫开始对艾希进行攻击并使其立即中毒。中毒状态会维持 2 秒钟，直到第 2 秒钟结束。在第 4 秒开始时，提莫再次攻击艾希，使得艾希获得另外 2 秒的中毒时间。所以最终输出 4 秒。

思路：就是考虑每次中毒的真是有效时间，所有次数累加起来即可，临界值要判断一下

```java
package com.qiming.algorithm.leetcode;

/**
 * 提莫攻击
 *
 * 在《英雄联盟》的世界中，有一个叫 “提莫” 的英雄，他的攻击可以让敌方英雄艾希（编者注：寒冰射手）进入中毒状态。现在，给出提莫对艾希的攻击时间序列和提莫攻击的中毒持续时间，你需要输出艾希的中毒状态总时长。
 * 你可以认为提莫在给定的时间点进行攻击，并立即使艾希处于中毒状态
 *
 * 输入: [1,4], 2  输出: 4  原因: 在第 1 秒开始时，提莫开始对艾希进行攻击并使其立即中毒。中毒状态会维持 2 秒钟，直到第 2 秒钟结束。
 * 在第 4 秒开始时，提莫再次攻击艾希，使得艾希获得另外 2 秒的中毒时间。所以最终输出 4 秒。
 *
 * 输入: [1,2], 2  输出: 3  原因: 在第 1 秒开始时，提莫开始对艾希进行攻击并使其立即中毒。中毒状态会维持 2 秒钟，直到第 2 秒钟结束。
 * 但是在第 2 秒开始时，提莫再次攻击了已经处于中毒状态的艾希。由于中毒状态不可叠加，提莫在第 2 秒开始时的这次攻击会在第 3 秒钟结束。所以最终输出 3。
 *
 *
 * 思路：就是考虑每次中毒的真是有效时间，所有次数累加起来即可，临界值要判断一下
 *
 */
public class TeemoAttacking {

  public int findPoisonedDuration(int[] timeSeries, int duration) {

    if (timeSeries.length == 0 || duration == 0) {
      return 0;
    }

    int result = 0;
    for (int i = 0; i < timeSeries.length - 1; i++) {
      int timeEve = 0;
      if (timeSeries[i] + duration - 1 < timeSeries[i+1]) {
        timeEve = duration;
      } else {
        timeEve = timeSeries[i+1] - timeSeries[i];
      }
      result = result + timeEve;
    }

    return result + duration;
  }

}
```

#### 81.验证二叉树的前序序列化（LC）

题目：序列化二叉树的一种方法是使用前序遍历。当我们遇到一个非空节点时，我们可以记录下这个节点的值。如果它是一个空节点，我们可以使用一个标记值记录，例如 #。例如，上面的二叉树可以被序列化为字符串 "9,3,4,#,#,1,#,#,2,#,6,#,#"，其中 # 代表一个空节点，给定一串以逗号分隔的序列，验证它是否是正确的二叉树的前序序列化。编写一个在不重构树的条件下的可行算法。每个以逗号分隔的字符或为一个整数或为一个表示 null 指针的 '#' 。你可以认为输入格式总是有效的，例如它永远不会包含两个连续的逗号，比如 "1,,3" 。输入: "9,3,4,#,#,1,#,#,2,#,6,#,#" 输出: true  输入: "1,#"  输出: false  输入: "9,#,#,1" 输出: false

思路：读取到的结构只要符合二叉树性质而且不会在未读完之前就满足leaves = nodes + 1（完整的二叉树）即可

```java
package com.qiming.algorithm.leetcode;

/**
 * 验证二叉树的前序序列化
 *
 * 序列化二叉树的一种方法是使用前序遍历。当我们遇到一个非空节点时，我们可以记录下这个节点的值。如果它是一个空节点，我们可以使用一个标记值记录，例如 #。
 *
 *      _9_
 *     /   \
 *    3     2
 *   / \   / \
 *  4   1  #  6
 * / \ / \   / \
 * # # # #   # #
 *
 * 例如，上面的二叉树可以被序列化为字符串 "9,3,4,#,#,1,#,#,2,#,6,#,#"，其中 # 代表一个空节点，给定一串以逗号分隔的序列，验证它是否是正确的二叉树的前序序列化。编写一个在不重构树的条件下的可行算法。
 *
 * 每个以逗号分隔的字符或为一个整数或为一个表示 null 指针的 '#' 。你可以认为输入格式总是有效的，例如它永远不会包含两个连续的逗号，比如 "1,,3" 。
 *
 * 输入: "9,3,4,#,#,1,#,#,2,#,6,#,#" 输出: true  输入: "1,#"  输出: false  输入: "9,#,#,1" 输出: false
 *
 * 思路：读取到的结构只要符合二叉树性质而且不会在未读完之前就满足leaves = nodes + 1（完整的二叉树）即可
 *
 *
 */
public class VerifyPreorderSerializationOfABinaryTree {

  public boolean isValidSerialization(String preorder) {

    int leaves = 0;
    int node = 0;
    String str[] = preorder.split(",");

    for (int i = 0; i < str.length; i++) {
      if (str[i].equals("#")) {
        leaves++;
      } else {
        node++;
      }
      //叶结点的个数
      if (leaves > node + 1) {
        return false;
      }
      if (leaves == node + 1 && i < str.length - 1) {
        return false;
      }
    }

    if (leaves != node + 1) {
      return false;
    } else {
      return true;
    }

  }

}

```

#### 82.岛屿的周长（LC）

题目：给定一个包含 0 和 1 的二维网格地图，其中 1 表示陆地 0 表示水域。网格中的格子水平和垂直方向相连（对角线方向不相连）。整个网格被水完全包围，但其中恰好有一个岛屿（或者说，一个或多个表示陆地的格子相连组成的岛屿）。岛屿中没有“湖”（“湖” 指水域在岛屿内部且不和岛屿周围的水相连）。格子是边长为 1 的正方形。网格为长方形，且宽度和高度均不超过 100 。计算这个岛屿的周长。

思路：计算北面+西面的长度后再乘以2就行了

```java
package com.qiming.algorithm.leetcode;

/**
 * 岛屿的周长
 *
 * 给定一个包含 0 和 1 的二维网格地图，其中 1 表示陆地 0 表示水域。网格中的格子水平和垂直方向相连（对角线方向不相连）。整个网格被水完全包围，但其中恰好有一个岛屿（或者说，一个或多个表示陆地的格子相连组成的岛屿）。
 * 岛屿中没有“湖”（“湖” 指水域在岛屿内部且不和岛屿周围的水相连）。格子是边长为 1 的正方形。网格为长方形，且宽度和高度均不超过 100 。计算这个岛屿的周长。
 *
 * 输入:
 * [[0,1,0,0],
 *  [1,1,1,0],
 *  [0,1,0,0],
 *  [1,1,0,0]]
 *
 * 输出: 16
 *
 * 思路：计算北面+西面的长度后再乘以2就行了
 *
 */
public class IslandPerimeter {

  public int islandPerimeter(int[][] grid) {

    int result = 0;
    for (int i = 0; i < grid.length; i++) {
      for (int j = 0; j < grid[0].length; j++) {
        if (grid[i][j] == 1) {
          //满足连续
          if (j == 0 || grid[i][j-1] == 0) {
            result++;
          }
          //满足连续
          if (i == 0 || grid[i-1][j] == 0) {
            result++;
          }
        }
      }
    }
    return result * 2;
  }


}

```

#### 83.二叉树的最大深度（LC）

题目：给定一个二叉树，找出其最大深度。二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。给定二叉树 [3,9,20,null,null,15,7]  返回它的最大深度 3 。

思路：层次遍历辅助队列

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 二叉树的最大深度
 *
 * 给定一个二叉树，找出其最大深度。二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。
 *
 * 给定二叉树 [3,9,20,null,null,15,7]  返回它的最大深度 3 。
 *
 * 思路：层次遍历辅助队列
 *
 */
public class MaximumDepthOfBinaryTree {

  public static void main(String[] args) {
    TreeNodeMaximumDepthOfBinaryTree p1 = new TreeNodeMaximumDepthOfBinaryTree(0);
    TreeNodeMaximumDepthOfBinaryTree p2 = new TreeNodeMaximumDepthOfBinaryTree(2);
    TreeNodeMaximumDepthOfBinaryTree p3 = new TreeNodeMaximumDepthOfBinaryTree(4);
    TreeNodeMaximumDepthOfBinaryTree p4 = new TreeNodeMaximumDepthOfBinaryTree(1);
    TreeNodeMaximumDepthOfBinaryTree p5 = new TreeNodeMaximumDepthOfBinaryTree(3);
    TreeNodeMaximumDepthOfBinaryTree p6 = new TreeNodeMaximumDepthOfBinaryTree(-1);
    TreeNodeMaximumDepthOfBinaryTree p7 = new TreeNodeMaximumDepthOfBinaryTree(5);
    TreeNodeMaximumDepthOfBinaryTree p8 = new TreeNodeMaximumDepthOfBinaryTree(1);
    TreeNodeMaximumDepthOfBinaryTree p9 = new TreeNodeMaximumDepthOfBinaryTree(6);
    TreeNodeMaximumDepthOfBinaryTree p10 = new TreeNodeMaximumDepthOfBinaryTree(8);
    p1.left = p2;
    p1.right = p3;
    p2.left = p4;
    p3.left = p5;
    p3.right = p6;
    p4.left = p7;
    p4.right = p8;
    p5.right = p9;
    p6.right = p10;
    System.out.println(new MaximumDepthOfBinaryTree().maxDepth(p1));
  }

  public int maxDepth(TreeNodeMaximumDepthOfBinaryTree root) {

    if (root == null) {
      return 0;
    }
    Queue<TreeNodeMaximumDepthOfBinaryTree> queue = new LinkedList();
    queue.offer(root);
    int result = 0;
    while (!queue.isEmpty()) {
      int size = queue.size(); //细节细节
      for (int i = 0; i < size; i++) {
        TreeNodeMaximumDepthOfBinaryTree p = queue.poll();
        if (p.left != null) {
          queue.offer(p.left);
        }
        if (p.right != null) {
          queue.offer(p.right);
        }
      }
      result++;
    }

    return result;

  }

}

class TreeNodeMaximumDepthOfBinaryTree {
  int val;
  TreeNodeMaximumDepthOfBinaryTree left;
  TreeNodeMaximumDepthOfBinaryTree right;
  TreeNodeMaximumDepthOfBinaryTree(int x) { val = x; }
}

```

#### 84.K 次取反后最大化的数组和（LC）

题目：给定一个整数数组 A，我们只能用以下方法修改该数组：我们选择某个个索引 i 并将 A[i] 替换为 -A[i]，然后总共重复这个过程 K 次。（我们可以多次选择同一个索引 i。）以这种方式修改数组后，返回数组可能的最大和。示例 1：输入：A = [4,2,3], K = 1 输出：5 解释：选择索引 (1,) ，然后 A 变为 [4,-2,3]。

思路：辅助数组，套路的，主要是要发现最小值和进行取反要想清楚

```java
package com.qiming.algorithm.leetcode;

/**
 * K 次取反后最大化的数组和
 *
 * 给定一个整数数组 A，我们只能用以下方法修改该数组：我们选择某个个索引 i 并将 A[i] 替换为 -A[i]，然后总共重复这个过程 K 次。（我们可以多次选择同一个索引 i。）
 * 以这种方式修改数组后，返回数组可能的最大和。
 *
 * 示例 1：输入：A = [4,2,3], K = 1 输出：5 解释：选择索引 (1,) ，然后 A 变为 [4,-2,3]。
 * 示例 2：输入：A = [3,-1,0,2], K = 3 输出：6 解释：选择索引 (1, 2, 2) ，然后 A 变为 [3,1,0,2]。
 *
 *
 * 思路：辅助数组，套路的，主要是要发现最小值和进行取反要想清楚
 *
 */
public class MaximizeSumOfArrayAfterKNegations {

  public int largestSumAfterKNegations(int[] A, int K) {

    int[] number = new int[201]; //-100到100的所有数

    for (int t : A) {
      number[t + 100]++;
    }
    int i = 0;
    while (K > 0) {
      while (number[i] == 0) {
        i++;
      }
      number[i]--;
      number[200-i]++; //相反个数+1
      if (i > 100) { //若原最小数索引>100,则新的最小数索引应为200-i.(索引即number[]数组的下标)
        i = 200 - i;
      }
      K--;
    }

    int sum = 0;
    for (int j = i; j < number.length; j++) {
      sum += (j-100)*number[j]; //j-100是数字大小
    }

    return sum;
  }

}

```

#### 85.设计哈希映射（LC）

题目：不使用任何内建的哈希表库设计一个哈希映射，具体地说，你的设计应该包含以下的功能，put(key, value)：向哈希映射中插入(键,值)的数值对。如果键对应的值已经存在，更新这个值。get(key)：返回给定的键所对应的值，如果映射中不包含这个键，返回-1。remove(key)：如果映射中存在这个键，删除这个数值对

思路：还蛮重要的，hashmap的实现，取巧的是用全值数组，还有就是正常写法，不考虐扩容，用双端队列(这才是要掌握的方法)

```java
package com.qiming.algorithm.leetcode;

import java.util.Arrays;

/**
 * 设计哈希映射
 *
 * 不使用任何内建的哈希表库设计一个哈希映射，具体地说，你的设计应该包含以下的功能
 *
 * put(key, value)：向哈希映射中插入(键,值)的数值对。如果键对应的值已经存在，更新这个值。get(key)：返回给定的键所对应的值，如果映射中不包含这个键，返回-1。
 *
 * remove(key)：如果映射中存在这个键，删除这个数值对
 *
 * MyHashMap hashMap = new MyHashMap();
 * hashMap.put(1, 1);          
 * hashMap.put(2, 2);        
 * hashMap.get(1);            // 返回 1
 * hashMap.get(3);            // 返回 -1 (未找到)
 * hashMap.put(2, 1);         // 更新已有的值
 * hashMap.get(2);            // 返回 1
 * hashMap.remove(2);         // 删除键为2的数据
 * hashMap.get(2);            // 返回 -1 (未找到)
 *
 * 思路：还蛮重要的，hashmap的实现，取巧的是用全值数组，还有就是正常写法，不考虐扩容，用双端队列(这才是要掌握的方法)
 *
 */
public class DesignHashMap {

//  int table[];

  class Node {
    int key, value;
    Node pre, next;

    Node (int key, int value) {
      this.key = key;
      this.value = value;
    }
  }

  private Node[] data;
  private int length = 100;

  /** Initialize your data structure here. */
  public DesignHashMap() {
//    table = new int[1000000];
//    Arrays.fill(table, -1);
    data = new Node[length];
  }

  /** value will always be non-negative. */
  public void put(int key, int value) {
//    table[key] = value;
    int index = key % length;
    Node curr = data[index];
    if (curr == null) {
      Node node = new Node(key, value);
      data[index] = node;
      return;
    } else {
      while (true) {
        if (curr.key == key) {
          curr.value = value;
          return;
        }
        if (curr.next == null) {
          Node node = new Node(key, value);
          node.pre = curr;
          curr.next = node;
          return;
        } else {
          //一直找到最后
          curr = curr.next;
        }
      }
    }

  }

  /** Returns the value to which the specified key is mapped, or -1 if this map contains no mapping for the key */
  public int get(int key) {
//    return table[key];
    int index = key % length;
    Node curr = data[index];
    while (curr != null) {
      if (curr.key == key) {
        return curr.value;
      }
      curr = curr.next;
    }
    return -1;
  }

  /** Removes the mapping of the specified value key if this map contains a mapping for the key */
  public void remove(int key) {
//    table[key] = -1;
    int index = key % length;
    Node curr = data[index];
    //就是第一个结点
    if (curr != null && curr.key == key) {
      Node next = curr.next;
      if (next != null) {
        next.pre = null;
      }
      data[index] = next;
      return;
    }
    while (curr != null) {
      if (curr.key == key) {
        Node next = curr.next;
        Node pre = curr.pre;
        if (next != null) {
          next.pre = pre;
        }
        if (pre != null) {
          pre.next = next;
        }
        return;
      }
      curr = curr.next;
    }

  }

}

/**
 * Your DesignHashMap object will be instantiated and called as such:
 * DesignHashMap obj = new DesignHashMap();
 * obj.put(key,value);
 * int param_2 = obj.get(key);
 * obj.remove(key);
 */
```

#### 86.完全二叉树的节点个数（LC）

题目：给出一个完全二叉树，求出该树的节点个数。

思路：递归和非递归的方式都可以去解，然后是充分利用完全二叉树性质的解，详看代码

```java
package com.qiming.algorithm.leetcode;

import java.util.LinkedList;
import java.util.Queue;

/**
 * 完全二叉树的节点个数
 *
 * 给出一个完全二叉树，求出该树的节点个数。
 *
 * 思路，递归和非递归的方式都可以去解，然后是充分利用完全二叉树性质的解，
 *
 * 它是一棵空树或者它的叶子节点只出在最后两层，若最后一层不满则叶子节点只在最左侧。再来回顾一下满二叉的节点个数怎么计算，如果满二叉树的层数为h，则总节点数为：2^h - 1.
 * 么我们来对root节点的左右子树进行高度统计，分别记为left和right,有以下两种结果：
 * 1、left == right。这说明，左子树一定是满二叉树，因为节点已经填充到右子树了，左子树必定已经填满了。所以左子树的节点总数我们可以直接得到，是2^left - 1，加上当前这个root节点，则正好是2^left。再对右子树进行递归统计。
 * 2、left != right。说明此时最后一层不满，但倒数第二层已经满了，可以直接得到右子树的节点个数。同理，右子树节点+root节点，总数为2^right。再对左子树进行递归查找。
 *
 * 关于如何计算二叉树的层数，可以利用下面的递归来算，当然对于完全二叉树，可以利用其特点，不用递归直接算，就是左子树的深度
 *
 * 如何计算2^left，最快的方法是移位计算，因为运算符的优先级问题，记得加括号哦。速度非常快
 *
 *
 */
public class CountCompleteTreeNodes {

  public int countNodes(TreeNodeCountCompleteTreeNodes root) {

//    if (root == null) {
//      return 0;
//    }
//
//    Queue<TreeNodeCountCompleteTreeNodes> queue = new LinkedList();
//    queue.offer(root);
//    int result = 0;
//    while (!queue.isEmpty()) {
//      int size = queue.size();
//      for (int i = 0; i < size; i++) {
//        TreeNodeCountCompleteTreeNodes p = queue.poll();
//        if (p.left != null) {
//          queue.offer(p.left);
//        }
//        if (p.right != null) {
//          queue.offer(p.right);
//        }
//      }
//      result += size;
//    }
//
//    return result;
    if(root == null){
      return 0;
    }
    int left = countLevel(root.left);
    int right = countLevel(root.right);
    if(left == right){
      return countNodes(root.right) + (1<<left);
    }else{
      return countNodes(root.left) + (1<<right);
    }


  }

  private int countLevel(TreeNodeCountCompleteTreeNodes root){
    int level = 0;
    while(root != null){
      level++;
      root = root.left;
    }
    return level;
  }


}


class TreeNodeCountCompleteTreeNodes {

  int val;
  TreeNodeCountCompleteTreeNodes left;
  TreeNodeCountCompleteTreeNodes right;
  TreeNodeCountCompleteTreeNodes(int x) {val = x;};

}
```

#### 87.圆圈中最后剩下的数字（LC）

题目：0,1,,n-1这n个数字排成一个圆圈，从数字0开始，每次从这个圆圈里删除第m个数字。求出这个圆圈里剩下的最后一个数字。例如，0、1、2、3、4这5个数字组成一个圆圈，从数字0开始每次删除第3个数字，则删除的前4个数字依次是2、0、4、1，因此最后剩下的数字是3。

思路：有点那什么意思，链表的方式是超时的！！！但是这里要说的是怎么保证是环，就是如果不是第m个元素，就将数值添加到arraylist的尾部，每次都移除第一个元素，这样还是能保证环的，这道题是一道约瑟夫环的问题，我们可以用数学的方法来做，迭代和递归，f(n,m)=（f(n−1,m)+m)modn，且f(1,m)=0

```java
package com.qiming.algorithm.leetcode;

import java.util.ArrayList;
import java.util.List;

/**
 * 圆圈中最后剩下的数字
 *
 * 0,1,,n-1这n个数字排成一个圆圈，从数字0开始，每次从这个圆圈里删除第m个数字。求出这个圆圈里剩下的最后一个数字。
 * 例如，0、1、2、3、4这5个数字组成一个圆圈，从数字0开始每次删除第3个数字，则删除的前4个数字依次是2、0、4、1，因此最后剩下的数字是3。
 *
 * 思路：有点那什么意思，链表的方式是超时的！！！但是这里要说的是怎么保证是环，就是如果不是第m个元素，就将数值添加到arraylist的尾部，每次都移除第一个元素，这样还是能保证环的
 *
 * 这道题是一道约瑟夫环的问题，我们可以用数学的方法来做，迭代和递归
 *
 * f(n,m)=（f(n−1,m)+m)modn，且f(1,m)=0
 *
 */
public class LastLeftNum {

  public int lastRemaining(int n, int m) {
    int flag = 0;
    for (int i = 2; i <= n; i++) {
      flag = (flag + m) % i;
      //动态规划的思想，将f(n,m)替换成flag存储
    }
    return flag;

    //通过举例可以得出第一次删除的数字下标为(m-1)%n记为c，之后每一次删除的数字下标均为(c+m-1)%list.size()
//    if(n==0||m==0)
//      return -1;
//    List<Integer> list=new ArrayList<>();
//    for(int i=0;i<n;i++)
//      list.add(i);
//    int c=(m-1)%n;
//    while(list.size()!=1) {
//      list.remove(c);
//      c=(c+m-1)%list.size();
//    }
//    return list.get(0);


  }


}

```

#### 88.删除回文子序列（LC）

题目：给你一个字符串 s，它仅由字母 'a' 和 'b' 组成。每一次删除操作都可以从 s 中删除一个回文 子序列。返回删除给定字符串中所有字符（字符串为空）的最小删除次数。「子序列」定义：如果一个字符串可以通过删除原字符串某些字符而不改变原字符顺序得到，那么这个字符串就是原字符串的一个子序列。

思路：这tm是个脑筋急转弯啊，子序列并不一定是连续的。。。如果字符串不是一个回文，就一口气先删a，再删b，如果是，删一次就行了

```java
package com.qiming.algorithm.leetcode;

/**
 * 删除回文子序列
 *
 * 给你一个字符串 s，它仅由字母 'a' 和 'b' 组成。每一次删除操作都可以从 s 中删除一个回文 子序列。返回删除给定字符串中所有字符（字符串为空）的最小删除次数。
 *
 * 「子序列」定义：如果一个字符串可以通过删除原字符串某些字符而不改变原字符顺序得到，那么这个字符串就是原字符串的一个子序列。
 *
 * 「回文」定义：如果一个字符串向后和向前读是一致的，那么这个字符串就是一个回文。
 *
 * 示例 1： 输入：s = "ababa" 输出：1 解释：字符串本身就是回文序列，只需要删除一次。
 * 示例 2： 输入：s = "abb"  输出：2  解释："abb" -> "bb" -> "".  先删除回文子序列 "a"，然后再删除 "bb"。
 * 示例 3： 输入：s = "baabb" 输出：2 解释："baabb" -> "b" -> "". 先删除回文子序列 "baab"，然后再删除 "b"。
 * 示例 4： 输入：s = "" 输出：0
 *
 * 0 <= s.length <= 1000  s 仅包含字母 'a'  和 'b'
 *
 * 思路：这tm是个脑筋急转弯啊，子序列并不一定是连续的。。。如果字符串不是一个回文，就一口气先删a，再删b，如果是，删一次就行了
 *
 */
public class RemovePalindromicSubsequences {

  public int removePalindromeSub(String s) {

    if (s.length() == 0) {
      return 0;
    }

    int left = 0;
    int right = s.length() - 1;
    while (left < right) {
      if (s.charAt(left) != s.charAt(right)) {
        return 2;
      }
      left++;
      right--;
    }
    return 1;

  }

}

```

#### 89.从尾到头打印链表（LC）

题目：输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

思路：用栈

```java
package com.qiming.algorithm.leetcode;

import java.util.Stack;

/**
 * 从尾到头打印链表
 *
 * 输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。
 *
 * 思路：用栈
 *
 */
public class PrintLinkedListReversal {

  public int[] reversePrint(ListNodePrintLinkedListReversal head) {

    Stack<Integer> stack = new Stack();
    while (head != null) {
      stack.push(head.val);
      head = head.next;
    }
    int result[] = new int[stack.size()];
    for (int i = 0; i < result.length; i++) {
      result[i] = stack.pop();
    }
    return result;

  }

}

class ListNodePrintLinkedListReversal {
  int val;
  ListNodePrintLinkedListReversal next;
  ListNodePrintLinkedListReversal(int x) {val = x;}

}
```

#### 90.阶乘后的零（LC）

题目：给定一个整数 n，返回 n! 结果尾数中零的数量，输入: 3  输出: 0  解释: 3! = 6, 尾数中没有零。  输入: 5  输出: 1  解释: 5! = 120, 尾数中有 1 个零. 说明: 你算法的时间复杂度应为 O(log n) 。

思路：数学题，直接看代码吧

```java
package com.qiming.algorithm.leetcode;

/**
 * 阶乘后的零
 *
 * 给定一个整数 n，返回 n! 结果尾数中零的数量。
 *
 * 输入: 3  输出: 0  解释: 3! = 6, 尾数中没有零。  输入: 5  输出: 1  解释: 5! = 120, 尾数中有 1 个零. 说明: 你算法的时间复杂度应为 O(log n) 。
 *
 * 思路：数学题，就是算5的个数，因为只有5*2为10了才会在后面加零，而有5的时候必出现过2的因子，因为2是2个2个一走，5是5个5个一走，但是硬算每个值的5的个数会超时
 *
 * 所以还要再想，5每隔5间隔会出现一次，而5*5=25间隔会出现两次，5*5*5=125间隔会出现三次，所以其实最后算的是N/5+N/25+N/125+...但是分母太大会溢出，N= N/5计算
 *
 */
public class FactorialTrailingZeroes {

  public int trailingZeroes(int n) {

    int count = 0;
    while (n > 0) {
      count += n / 5;
      n = n / 5;
    }
    return count;
  }

}

```

#### 91.合并二叉树（LC）

题目：给定两个二叉树，想象当你将它们中的一个覆盖到另一个上时，两个二叉树的一些节点便会重叠。你需要将他们合并为一个新的二叉树。合并的规则是如果两个节点重叠，那么将他们的值相加作为节点合并后的新值，否则不为 NULL 的节点将直接作为新二叉树的节点。

思路：递归，清晰一点，迭代需要用到辅助栈

```java
package com.qiming.algorithm.leetcode;

/**
 * 合并二叉树
 *
 * 给定两个二叉树，想象当你将它们中的一个覆盖到另一个上时，两个二叉树的一些节点便会重叠。你需要将他们合并为一个新的二叉树。合并的规则是如果两个节点重叠，
 * 那么将他们的值相加作为节点合并后的新值，否则不为 NULL 的节点将直接作为新二叉树的节点。
 *
 * 思路：递归，清晰一点，迭代需要用到辅助栈
 *
 */
public class MergeTwoBinaryTrees {

  public TreeNodeMergeTwoBinaryTrees mergeTrees(TreeNodeMergeTwoBinaryTrees t1, TreeNodeMergeTwoBinaryTrees t2) {

    if (t1 == null) {
      return t2;
    }

    if (t2 == null) {
      return t1;
    }
    //这边也可以不new，直接用t1
    TreeNodeMergeTwoBinaryTrees root = new TreeNodeMergeTwoBinaryTrees(t1.val + t2.val);
    root.left = mergeTrees(t1.left, t2.left);
    root.right = mergeTrees(t1.right, t2.right);

    return root;

  }

}

class TreeNodeMergeTwoBinaryTrees {

  int val;
  TreeNodeMergeTwoBinaryTrees left;
  TreeNodeMergeTwoBinaryTrees right;
  TreeNodeMergeTwoBinaryTrees(int x) { val = x;}

}

```

#### 92.2的幂（LC）

题目：给定一个整数，编写一个函数来判断它是否是 2 的幂次方。

思路：int类型，移位

```java
package com.qiming.algorithm.leetcode;

/**
 * 2的幂
 *
 * 给定一个整数，编写一个函数来判断它是否是 2 的幂次方。
 *
 * 思路：int类型，移位
 *
 */
public class PowerOfTwo {

  public static void main(String[] args) {

    int k = 1;
    for (int i = 0; i < 30; i++) {
      k = k << 1;
    }
    System.out.println(k);

  }

  public boolean isPowerOfTwo(int n) {

    int k = 1;
    for (int i = 0; i < 31; i++) {
      //注意k移位和判断有前后顺序的哦，影响i的上限是30还是31
      if (k == n) {
        return true;
      }
      k = k << 1;
    }
    return false;
  }

}

```

#### 93.跳跃游戏（LC）

题目：给定一个非负整数数组，你最初位于数组的第一个位置。数组中的每个元素代表你在该位置可以跳跃的最大长度。判断你是否能够到达最后一个位置。示例 1: 输入: [2,3,1,1,4] 输出: true 解释: 我们可以先跳 1 步，从位置 0 到达 位置 1, 然后再从位置 1 跳 3 步到达最后一个位置。输入: [3,2,1,0,4] 输入: [3,2,1,0,4] 解释: 无论怎样，你总会到达索引为 3 的位置。但该位置的最大跳跃长度是 0 ， 所以你永远不可能到达最后一个位置。

思路：动态规划，这个可以在数据结构和算法中看看


```java
package com.qiming.algorithm.leetcode;

/**
 * 跳跃游戏
 *
 * 给定一个非负整数数组，你最初位于数组的第一个位置。数组中的每个元素代表你在该位置可以跳跃的最大长度。
 *
 * 判断你是否能够到达最后一个位置。
 *
 * 示例 1: 输入: [2,3,1,1,4] 输出: true 解释: 我们可以先跳 1 步，从位置 0 到达 位置 1, 然后再从位置 1 跳 3 步到达最后一个位置。
 *
 * 输入: [3,2,1,0,4] 输入: [3,2,1,0,4] 解释: 无论怎样，你总会到达索引为 3 的位置。但该位置的最大跳跃长度是 0 ， 所以你永远不可能到达最后一个位置。
 *
 * 思路：动态规划，这个可以在数据结构和算法中看看
 *
 */
public class JumpGame {

  public static void main(String[] args) {

    int test[] = {2,3,1,1,4};
    new JumpGame().canJump(test);
  }


  //还是得用贪心或者动态规划
  public boolean canJump(int[] nums) {
//    if (nums.length == 1) {
//      return true;
//    }
//    return helper(nums, 0, nums.length);
    int lastPos = nums.length - 1;
    for (int i = nums.length - 1; i >= 0; i--) {
      if (i + nums[i] >= lastPos) {
        lastPos = i;
      }
    }
    return lastPos == 0;
  }

/**
 * 错误案例
 */
//  public boolean canJump(int[] nums) {
//    if (nums.length == 1) {
//      return true;
//    }
//    return helper(nums, 0, nums.length);
//  }
//
//  private boolean helper (int[] nums, int start, int limit) {
//    for (int i = start; i < nums.length - 1; i++) {
//      //代表会有问题，迭代去检查，这样写会有问题[3,0,8,2,0,0,1]就过不鸟
//      if (nums[i] == 0) {
//        return false;
//      }
//      if (nums[i] < limit - 1) {
//        boolean result = helper(nums, start + 1, limit - 1);
//        if (result) {
//          return true;
//        }
//      } else {
//        return true;
//      }
//    }
//    return false;
//  }

}

```
#### 94.有序数组中出现次数超过25%的元素（LC）

题目：给你一个非递减的 有序 整数数组，已知这个数组中恰好有一个整数，它的出现次数超过数组元素总数的 25%。请你找到并返回这个整数。输入：arr = [1,2,2,6,6,6,6,7,10] 输出：6    1 <= arr.length <= 10^4   0 <= arr[i] <= 10^5

思路：二分查找不太好理解，正常遍历一遍写吧

```java
package com.qiming.algorithm.leetcode;

/**
 * 有序数组中出现次数超过25%的元素
 *
 * 给你一个非递减的 有序 整数数组，已知这个数组中恰好有一个整数，它的出现次数超过数组元素总数的 25%。请你找到并返回这个整数
 *
 * 输入：arr = [1,2,2,6,6,6,6,7,10] 输出：6    1 <= arr.length <= 10^4   0 <= arr[i] <= 10^5
 *
 * 思路：二分查找不太好理解，正常遍历一遍写吧
 *
 */
public class ElementAppearingMoreThan25InSortedArray {

  public int findSpecialInteger(int[] arr) {

    if (arr.length == 1) {
      return arr[0];
    }

    int limit = arr.length / 4 + 1;

    int result = 1;
    for (int i = 0; i < arr.length; i++) {
      if (arr[i] == arr[i+1]) {
        result++;
        // >= 是会出现[1,1]的情况，不用等于
        if (result >= limit) {
          return arr[i];
        }
      } else {
        result = 1;
      }
    }

    return -1;
  }

}

```

#### 95.整数替换（LC）

题目：给定一个正整数 n，你可以做如下操作：1. 如果 n 是偶数，则用 n / 2替换 n。 2. 如果 n 是奇数，则可以用 n + 1或n - 1替换 n。 n 变为 1 所需的最小替换次数是多少？示例 1: 输入: 8  输出: 3  解释: 8 -> 4 -> 2 -> 1  示例 2: 输入: 7  输出: 4  解释: 7 -> 8 -> 4 -> 2 -> 1 或 7 -> 6 -> 3 -> 2 -> 1

思路：一个是递归，还有一个非递归看下代码详解

```java
package com.qiming.algorithm.leetcode;

/**
 * 整数替换
 *
 * 给定一个正整数 n，你可以做如下操作：1. 如果 n 是偶数，则用 n / 2替换 n。 2. 如果 n 是奇数，则可以用 n + 1或n - 1替换 n。 n 变为 1 所需的最小替换次数是多少？
 *
 * 示例 1: 输入: 8  输出: 3  解释: 8 -> 4 -> 2 -> 1  示例 2: 输入: 7  输出: 4  解释: 7 -> 8 -> 4 -> 2 -> 1 或 7 -> 6 -> 3 -> 2 -> 1
 *
 * 思路：递归可以解，比较快的方法是位运算。当 n 为偶数时，不用犹豫，直接自除 2 即可。当 n 为奇数时，选择对 n 进行 +1 还是 -1 取决于哪一种运算能够带来更多的 **相连最低位的二进制 0 **，
 * 因为二进制最低位为 0，该数一定是偶数，是偶数则直接自除 2（即右移 1 位），相连的最低位的二进制 0 越多，代表右移的机会将会变多，n 也容易接近目标 1。奇数有两种情况：0bxx01 和 0bxx11，
 * 前者更适合做 -1 运算，因为 n 进行- 1 之后，二进制会比 +1 运算多一个 0； 后者(3 除外)更适合 +1 运算，因为 +1 会使两个二进制 1 都变成 0。以 0bxx11 结尾的数还有特殊的 3，特殊在 3+1=4(0bxx100)，
 * 比运算 -1 要多右移两次。
 *
 *
 */
public class IntegerReplacement {

  public int integerReplacement(int n) {
    int step = 0; //记录步数
    while (n != 1) {
      //偶数
      if ((n & 1) == 0) {
        n >>>= 1;
        step++;
      } else {
        //奇数
        n += ((n & 2) == 0 || n ==3) ? -1 : 1;
        step++;
      }
    }
    return step;

//    return f(n);

  }

  private int f(int n) {
    if (n == 1) {
      return 0;
    }
    if (n % 2 == 0) {
      // >>> 是无符号右移，二进制右移补零操作符，左操作数的值按右操作数指定的位数右移，移动得到的空位以零填充，如value >>> num中，num指定要移位值value 移动的位数
      return f(n >>> 1) + 1;
    } else {
      return Math.min(f(n-1), f(n+1)) + 1;
    }
  }

}

```

#### 96.至少是其他数字两倍的最大数（LC）

题目：在一个给定的数组nums中，总是存在一个最大元素 。查找数组中的最大元素是否至少是数组中每个其他数字的两倍。如果是，则返回最大元素的索引，否则返回-1。

思路：用辅助数组还不如线性扫描两次

```java
package com.qiming.algorithm;

import java.util.Arrays;

/**
 * 至少是其他数字两倍的最大数
 *
 * 在一个给定的数组nums中，总是存在一个最大元素 。查找数组中的最大元素是否至少是数组中每个其他数字的两倍。如果是，则返回最大元素的索引，否则返回-1。
 *
 * 示例 1: 输入: nums = [3, 6, 1, 0] 输出: 1 解释: 6是最大的整数, 对于数组中的其他整数,6大于数组中其他元素的两倍。6的索引是1, 所以我们返回1
 *
 * 示例 2: 输入: nums = [1, 2, 3, 4] 输出: -1 解释: 4没有超过3的两倍大, 所以我们返回 -1.
 *
 * nums 的长度范围在[1, 50]. 每个 nums[i] 的整数范围在 [0, 100].
 *
 * 思路：用辅助数组还不如线性扫描两次
 *
 */
public class LargestNumberAtLeastTwiceOfOthers {

  public int dominantIndex(int[] nums) {

    if (nums.length == 1) {
      return 0;
    }

    int mem[] = new int[101];

    Arrays.fill(mem, -1);

    for (int i = 0; i < nums.length; i++) {
      mem[nums[i]] = i;
    }

    int indexMax = mem.length - 1;

    while (mem[indexMax] == -1) {
      indexMax--;
    }
    int secondMax = indexMax - 1;

    while (mem[secondMax] == -1) {
      secondMax--;
    }

    if (indexMax >= secondMax * 2) {
      return mem[indexMax];
    } else {
      return -1;
    }

  }

}

```

#### 97.二叉树的最近公共祖先（LC）

题目：给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”

思路：用结点与父结点的关系用hashmap保存，这样，可以知道p的所有祖先set集合，再看q的祖先，第一个在set中出现的就是了

```java
package com.qiming.algorithm;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import java.util.Stack;

/**
 * 二叉树的最近公共祖先
 *
 * 给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，
 * 最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”
 *
 * 例如，给定如下二叉树:  root = [3,5,1,6,2,0,8,null,null,7,4]
 *
 * 输入: root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 1 输出: 3
 * 输入: root = [3,5,1,6,2,0,8,null,null,7,4], p = 5, q = 4 输出: 5
 *
 * 思路：用结点与父结点的关系用hashmap保存，这样，可以知道p的所有祖先set集合，再看q的祖先，第一个在set中出现的就是了
 *
 */
public class LowestCommonAncestorOfABinaryTree {

  public TreeNodeLowestCommonAncestorOfABinaryTree lowestCommonAncestor(TreeNodeLowestCommonAncestorOfABinaryTree root, TreeNodeLowestCommonAncestorOfABinaryTree p, TreeNodeLowestCommonAncestorOfABinaryTree q) {

    Stack<TreeNodeLowestCommonAncestorOfABinaryTree> stack = new Stack();
    //父亲字典
    Map<TreeNodeLowestCommonAncestorOfABinaryTree,TreeNodeLowestCommonAncestorOfABinaryTree> parent = new HashMap();

    parent.put(root, null);
    stack.push(root);

    //迭代直到即发现P，又发现Q
    while (!parent.containsKey(p) || !parent.containsKey(q)) {

      TreeNodeLowestCommonAncestorOfABinaryTree node = stack.pop();

      //保持记录父结点
      if (node.left != null) {
        parent.put(node.left, node);
        stack.push(node.left);
      }

      if (node.right != null) {
        parent.put(node.right, node);
        stack.push(node.right);
      }

    }

    //p结点的祖先集合
    Set<TreeNodeLowestCommonAncestorOfABinaryTree> ancestors = new HashSet<>();

    while (p != null) {
      ancestors.add(p);
      p = parent.get(p);
    }
    //遍历q的祖先，如果第一个在集合中出现，就是了
    while (!ancestors.contains(q)) {
      q = parent.get(q);
    }

    return q;

  }

}


class TreeNodeLowestCommonAncestorOfABinaryTree {

  int val;
  TreeNodeLowestCommonAncestorOfABinaryTree left;
  TreeNodeLowestCommonAncestorOfABinaryTree right;
  TreeNodeLowestCommonAncestorOfABinaryTree(int x) { val = x; }

}
```

#### 98.范围求和 II（LC）

题目：给定一个初始元素全部为 0，大小为 m*n 的矩阵 M 以及在 M 上的一系列更新操作。操作用二维数组表示，其中的每个操作用一个含有两个正整数 a 和 b 的数组表示，含义是将所有符合 0 <= i < a 以及 0 <= j < b 的元素 M[i][j] 的值都增加 1。在执行给定的一系列操作后，你需要返回矩阵中含有最大整数的元素个数。

思路：arr[0][0]始终会被加，所以一定是最大值，暴力破解可以，但肯定不好，其实关键点是左侧和上侧是固定死的了，也就是框的左上角是固定的，只要管右下角就行了，就是求矩形的交集，其实就是横竖的最小值

```java
package com.qiming.algorithm.leetcode;

/**
 * 范围求和 II
 *
 * 给定一个初始元素全部为 0，大小为 m*n 的矩阵 M 以及在 M 上的一系列更新操作。操作用二维数组表示，其中的每个操作用一个含有两个正整数 a 和 b 的数组表示，
 * 含义是将所有符合 0 <= i < a 以及 0 <= j < b 的元素 M[i][j] 的值都增加 1。在执行给定的一系列操作后，你需要返回矩阵中含有最大整数的元素个数。
 *
 * 输入:  m = 3, n = 3  operations = [[2,2],[3,3]]  解释:
 * 初始状态, M =
 * [[0, 0, 0],
 *  [0, 0, 0],
 *  [0, 0, 0]]
 *
 * 执行完操作 [2,2] 后, M =
 * [[1, 1, 0],
 *  [1, 1, 0],
 *  [0, 0, 0]]
 *
 * 执行完操作 [3,3] 后, M =
 * [[2, 2, 1],
 *  [2, 2, 1],
 *  [1, 1, 1]]
 *
 * M 中最大的整数是 2, 而且 M 中有4个值为2的元素。因此返回 4。 m 和 n 的范围是 [1,40000]。 a 的范围是 [1,m]，b 的范围是 [1,n]。 操作数目不超过 10000。
 *
 * 思路：arr[0][0]始终会被加，所以一定是最大值，暴力破解可以，但肯定不好，其实关键点是左侧和上侧是固定死的了，也就是框的左上角是固定的，只要管右下角就行了，
 * 就是求矩形的交集，其实就是横竖的最小值
 *
 *
 */
public class RangeAdditionII {

  public int maxCount(int m, int n, int[][] ops) {

    for (int[] op : ops) {
      m = Math.min(m, op[0]);
      n = Math.min(n, op[1]);
    }

    return m*n;

  }

}

```

#### 99.数组的度（LC）

题目：给定一个非空且只包含非负数的整数数组 nums, 数组的度的定义是指数组里任一元素出现频数的最大值。你的任务是找到与 nums 拥有相同大小的度的最短连续子数组，返回其长度。

思路：见代码正文吧，集合的使用，很赞

```java
package com.qiming.algorithm.leetcode;

import java.util.Collections;
import java.util.HashMap;
import java.util.Map;

/**
 * 数组的度
 *
 * 给定一个非空且只包含非负数的整数数组 nums, 数组的度的定义是指数组里任一元素出现频数的最大值。你的任务是找到与 nums 拥有相同大小的度的最短连续子数组，返回其长度。
 *
 * 示例 1: 输入: [1, 2, 2, 3, 1] 输出: 2 输入数组的度是2，因为元素1和2的出现频数最大，均为2.连续子数组里面拥有相同度的有如下所示:[1, 2, 2, 3, 1], [1, 2, 2, 3], [2, 2, 3, 1], [1, 2, 2], [2, 2, 3], [2, 2]
 * 最短连续子数组[2, 2]的长度为2，所以返回2.
 *
 * 思路：具有度数d的数组必须有一些元素x出现d次，如果某些子数组具有相同的度数，那么某些元素x（出现d次）。最短的子数组是将从x的第一次出现到最后一次出现的数组
 * 对于给定数组中的每个元素，让我们知道left是它的第一次出现的索引，right是它的最后一次出现的索引，例如，当nums = [1,2,3,2,5]时，left[2]=1和right[2]=3
 * 然后，对于出现次数最多的每个元素x，right[x]-left[x] + 1 将是我们的候选答案，取最小值
 *
 */
public class DegreeOfAnArray {

  public int findShortestSubArray(int[] nums) {

    Map<Integer, Integer> left = new HashMap(), right = new HashMap(), count = new HashMap();
    for (int i = 0; i < nums.length; i++) {
      int x = nums[i];
      if (left.get(x) == null) left.put(x, i);
      right.put(x, i);
      count.put(x, count.getOrDefault(x, 0) + 1);
    }

    int ans = nums.length;
    int degree = Collections.max(count.values());

    for (Integer x : count.keySet()) {
      if (count.get(x) == degree) {
        ans = Math.min(ans, right.get(x) - left.get(x) + 1);
      }
    }

    return ans;

  }

}

```

#### 100.单词子集（LC）

题目：我们给出两个单词数组 A 和 B。每个单词都是一串小写字母。现在，如果 b 中的每个字母都出现在 a 中，包括重复出现的字母，那么称单词 b 是单词 a 的子集。 例如，“wrr” 是 “warrior” 的子集，但不是 “world” 的子集。如果对 B 中的每一个单词 b，b 都是 a 的子集，那么我们称 A 中的单词 a 是通用的。你可以按任意顺序以列表形式返回 A 中所有的通用单词。

思路：正常思路肯定超时，n^3的写法，主要是变化这个通用单词的解法，当我们检验 "warrior" 是否是 B = ["wrr", "wa", "or"] 的超集时，我们以按照字母出现的最多次数将 B 中所有单词合并成一个单词 "arrow"，然后判断一次即可。

```java
package com.qiming.algorithm.leetcode;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.LinkedList;
import java.util.List;

/**
 * 单词子集
 *
 * 我们给出两个单词数组 A 和 B。每个单词都是一串小写字母。
 *
 * 现在，如果 b 中的每个字母都出现在 a 中，包括重复出现的字母，那么称单词 b 是单词 a 的子集。 例如，“wrr” 是 “warrior” 的子集，但不是 “world” 的子集。
 * 如果对 B 中的每一个单词 b，b 都是 a 的子集，那么我们称 A 中的单词 a 是通用的。你可以按任意顺序以列表形式返回 A 中所有的通用单词。
 *
 * 输入：A = ["amazon","apple","facebook","google","leetcode"], B = ["e","o"] 输出：["facebook","google","leetcode"]
 * 输入：A = ["amazon","apple","facebook","google","leetcode"], B = ["l","e"] 输出：["apple","google","leetcode"]
 * 输入：A = ["amazon","apple","facebook","google","leetcode"], B = ["e","oo"] 输出：["facebook","google"]
 *
 * 1 <= A.length, B.length <= 10000  1 <= A[i].length, B[i].length <= 10  A[i] 和 B[i] 只由小写字母组成 A[i]中所有的单词都是独一无二的，也就是说不存在 i != j 使得 A[i] == A[j]
 *
 * 思路：正常思路肯定超时，n^3的写法，主要是变化这个通用单词的解法，当我们检验 "warrior" 是否是 B = ["wrr", "wa", "or"] 的超集时，我们以按照字母出现的最多次数将 B 中所有单词合并成一个单词 "arrow"，然后判断一次即可。
 *
 */
public class WordSubsets {

  private int[] a = new int[26];
  private int[] b = new int[26];

  public static void main(String[] args) {

    String A[] = {"amazon","apple","facebook","google","leetcode"};
    String B[] = {"e","o"};

    new WordSubsets().wordSubsets(A, B);


  }

  public List<String> wordSubsets(String[] A, String[] B) {

    /**
     * 正常解法会超时
     */
//    List<String> result = new LinkedList<>();
//    for (int i = 0; i < A.length; i++) {
//      boolean canAdd = true;
//      for (int j = 0; j < B.length; j++) {
//        if (!isSubString(A[i], B[j])) {
//          canAdd = false;
//          break;
//        }
//      }
//      if (canAdd) result.add(A[i]);
//    }
//    return result;

    //就是拼成B的新单词，节省不少步骤，不用每个单词，每个单词去做了，合体
    int []bmax = count("");
    for (String b : B) {
      int[] bCount = count(b);
      for (int i = 0; i < 26; i++) {
        bmax[i] = Math.max(bmax[i], bCount[i]);
      }
    }
    List<String> ans = new LinkedList<>();
    for (String a : A) {
      int[] aCount = count(a);
      boolean canAdd = true;
      for (int i = 0; i < 26; i++) {
        if (aCount[i] < bmax[i]) {
          canAdd = false;
          break;
        }
      }
      if (canAdd) ans.add(a);
    }

    return ans;
  }

  private int[] count(String S) {
    int ans[] = new int[26];
    for (char c : S.toCharArray()) {
      ans[c-'a']++;
    }
    return ans;
  }

  private boolean isSubString(String A, String B) {
    Arrays.fill(a,0);
    Arrays.fill(b,0);
    for (int i = 0; i < A.length(); i++) {
      a[A.charAt(i)-'a']++;
    }
    for (int i = 0; i < B.length(); i++) {
      b[B.charAt(i)-'a']++;
    }
    for (int i = 0; i < b.length; i++) {
      if (b[i] != 0 && b[i] > a[i]) {
        return false;
      }
    }
    return true;
  }

}

```

#### 101.最长公共前缀（LC）

题目：编写一个函数来查找字符串数组中的最长公共前缀。如果不存在公共前缀，返回空字符串 ""。

思路：水平垂直法，分治法，二分查找法，[见](https://leetcode-cn.com/problems/longest-common-prefix/solution/zui-chang-gong-gong-qian-zhui-by-leetcode/)

```java
package com.qiming.algorithm.leetcode;

/**
 * 最长公共前缀
 *
 * 编写一个函数来查找字符串数组中的最长公共前缀。如果不存在公共前缀，返回空字符串 ""。
 *
 * 示例 1: 输入: ["flower","flow","flight"] 输出: "fl"
 * 示例 2: 输入: ["dog","racecar","car"] 输出: ""
 *
 * 所有输入只包含小写字母 a-z 。
 *
 * 思路：水平垂直法，分治法，二分查找法
 *
 */
public class LongestCommonPrefix {

  public static void main(String[] args) {
    String s1[] = {"flower","flow","flight"};
    String s2 = "abcg";
    new LongestCommonPrefix().longestCommonPrefix(s1);
    System.out.println("flow".indexOf("flower"));
  }

  public String longestCommonPrefix(String[] strs) {
    /**
     * 水平扫描法
     */
//    if (strs.length == 0) return "";
//    String prefix = strs[0];
//    for (int i = 1; i < strs.length; i++)
//      //成为公共前缀了就会为0，公共前缀index必从0开始
//      while (strs[i].indexOf(prefix) != 0) {
//        //每次缩小一个字符。再检查
//        prefix = prefix.substring(0, prefix.length() - 1);
//        //字符没有了，说明就没有公共前缀了，直接返回""
//        if (prefix.isEmpty()) return "";
//      }
//    return prefix;
    /**
     * 竖直扫描，每个字符上的每一列
     */
//    if (strs == null || strs.length == 0) return "";
//    for (int i = 0; i < strs[0].length() ; i++){
//      char c = strs[0].charAt(i);
//      for (int j = 1; j < strs.length; j ++) {
//        //判断条件是到底了和字符不一样
//        if (i == strs[j].length() || strs[j].charAt(i) != c)
//          return strs[0].substring(0, i);
//      }
//    }
//    return strs[0];
    /**
     * 分治，将单词分组，分别计算LCP，再合起来计算LCP，用了递归，并将问题分解成了两个字符串的比较
     */
//    if (strs == null || strs.length == 0) return "";
//    return longestCommonPrefix(strs, 0 , strs.length - 1);
    /**
     * 二分查找法，应用二分查找法找到所有字符串的公共前缀的最大长度 L
     */
    if (strs == null || strs.length == 0)
      return "";
    int minLen = Integer.MAX_VALUE;
    for (String str : strs)
      minLen = Math.min(minLen, str.length());
    int low = 1;
    int high = minLen;
    while (low <= high) {
      int middle = (low + high) / 2;
      if (isCommonPrefix(strs, middle))
        low = middle + 1;
      else
        high = middle - 1;
    }
    return strs[0].substring(0, (low + high) / 2);

  }

  /**
   * 分治法辅助函数
   * @param strs
   * @param l
   * @param r
   * @return
   */
  private String longestCommonPrefix(String[] strs, int l, int r) {
    if (l == r) {
      return strs[l];
    }
    else {
      int mid = (l + r)/2;
      String lcpLeft =   longestCommonPrefix(strs, l , mid);
      String lcpRight =  longestCommonPrefix(strs, mid + 1,r);
      return commonPrefix(lcpLeft, lcpRight);
    }
  }

  String commonPrefix(String left,String right) {
    int min = Math.min(left.length(), right.length());
    for (int i = 0; i < min; i++) {
      if ( left.charAt(i) != right.charAt(i) )
        return left.substring(0, i);
    }
    return left.substring(0, min);
  }

  /**
   * 二分查找法辅助函数
   * @param strs
   * @param len
   * @return
   */
  private boolean isCommonPrefix(String[] strs, int len){
    String str1 = strs[0].substring(0,len);
    for (int i = 1; i < strs.length; i++)
      if (!strs[i].startsWith(str1))
        return false;
    return true;
  }


}

```

#### 102.连续数列（LC）

题目：给定一个整数数组（有正数有负数），找出总和最大的连续数列，并返回总和。

思路：看代码正文

```java
package com.qiming.algorithm.leetcode;

/**
 * 连续数列
 *
 * 给定一个整数数组（有正数有负数），找出总和最大的连续数列，并返回总和。
 *
 * 示例：输入： [-2,1,-3,4,-1,2,1,-5,4]  输出： 6  解释： 连续子数组 [4,-1,2,1] 的和最大，为 6。
 *
 * 如果你已经实现复杂度为 O(n) 的解法，尝试使用更为精妙的分治法求解。
 *
 * 思路：大老板sum，和他的小喽啰b，以收割土地（数值）为生。b拿着sum的资产出门，遇到数值以后，b先看自己的资产，如果是正的，那么直接掠夺别人相加；
 * 如果是负的，就把自己的扔掉，让别人的数直接覆盖，因为如果相加的话，自己的负资产一定会拖后腿。然后带着自己的新资产情况回去和sum比较，
 * 来对比自己收割情况，情况好的话更新大老板数据，不好的话出去掠夺，以保持sum的总和是最大的。并且，b累计的和一直是连续计算的。这个弯子有点绕的
 *
 * 还有动态规划，dp[i]表示，从0到i，在包含元素i的情况下的最大值，找到dp中的最大值即可
 *
 */
public class ContiguousSequenceLCCI {

  public int maxSubArray(int[] nums) {

    int b = nums[0];
    int sum = b;
    for (int i = 1; i < nums.length; i++) {
      if (b < 0) {
        b = nums[i];
      } else {
        b += nums[i];
      }
      if (b > sum) {
        sum = b;
      }
    }
    return sum;
  }

}

```

#### 103.二叉树的直径（LC）

题目：给定一棵二叉树，你需要计算它的直径长度。一棵二叉树的直径长度是任意两个结点路径长度中的最大值。这条路径可能穿过根结点。

思路：小心陷阱，假设我们知道对于每个节点最长箭头距离分别为 L, RL,R，那么最优路径经过 L + R + 1 个节点。按照常用方法计算一个节点的深度：max(depth of node.left, depth of node.right) + 1。在计算的同时，经过这个节点的路径长度为 1 + (depth of node.left) + (depth of node.right) 。搜索每个节点并记录这些路径经过的点数最大值，期望长度是结果，要记得每个节点都要比较就行了

```java
package com.qiming.algorithm.leetcode;

/**
 * 二叉树的直径
 *
 * 给定一棵二叉树，你需要计算它的直径长度。一棵二叉树的直径长度是任意两个结点路径长度中的最大值。这条路径可能穿过根结点。
 *
 * 示例 :
 *           1
 *          / \
 *         2   3
 *        / \
 *       4   5
 * 返回 3, 它的长度是路径 [4,2,1,3] 或者 [5,2,1,3]。 注意：两结点之间的路径长度是以它们之间边的数目表示。
 *
 * 思路：小心陷阱，假设我们知道对于每个节点最长箭头距离分别为 L, RL,R，那么最优路径经过 L + R + 1 个节点。
 * 按照常用方法计算一个节点的深度：max(depth of node.left, depth of node.right) + 1。在计算的同时，经过这个节点的路径
 * 长度为 1 + (depth of node.left) + (depth of node.right) 。搜索每个节点并记录这些路径经过的点数最大值，期望长度是结果，要记得每个节点都要比较就行了
 *
 */
public class DiameterOfBinaryTree {

  int ans = 0;

  public int diameterOfBinaryTree(TreeNodeDiameterOfBinaryTree root) {

    /**
     * 最大路径一定经过根结点。。这个想法错了，最大值也可以在子树里面。。
     */
//    if (root == null) {
//      return 0;
//    }
//
//    int letfDepth = calDepth(root.left);
//    int rightDepth = calDepth(root.right);
//
//    return letfDepth + rightDepth;
    depth(root);
    return ans;


  }

  private int depth(TreeNodeDiameterOfBinaryTree root) {

    if (root == null) {
      return 0;
    }
    int L = depth(root.left);
    int R = depth(root.right);
    ans = Math.max(ans, L + R);
    return Math.max(L, R) + 1;
  }

  private int calDepth(TreeNodeDiameterOfBinaryTree root) {
    if (root == null) {
      return 0;
    }
    //也是种递归求高度的方法
    return Math.max(calDepth(root.left), calDepth(root.right)) + 1;
  }

}

class TreeNodeDiameterOfBinaryTree {

  int val;
  TreeNodeDiameterOfBinaryTree left;
  TreeNodeDiameterOfBinaryTree right;
  TreeNodeDiameterOfBinaryTree(int x) { val = x; }

}

```

#### 104.交替打印字符串（LC）

题目：编写一个可以从 1 到 n 输出代表这个数字的字符串的程序，但是：如果这个数字可以被 3 整除，输出 "fizz"。如果这个数字可以被 5 整除，输出 "buzz"。如果这个数字可以同时被 3 和 5 整除，输出 "fizzbuzz"。

思路：用信号量去做，注意number这个很关键，自我释放锁

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

#### 105.哲学家进餐（LC）

题目：设计一个进餐规则（并行算法）使得每个哲学家都不会挨饿；也就是说，在没有人知道别人什么时候想吃东西或思考的情况下，每个哲学家都可以在吃饭和思考之间一直交替下去。

思路：一个Semaphore和一个ReentrantLock数组

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

#### 106.从前序与中序遍历序列构造二叉树（LC）

题目：根据一棵树的前序遍历与中序遍历构造二叉树。

思路：首先，preorder 中的第一个元素一定是树的根，这个根又将 inorder 序列分成了左右两棵子树。现在我们只需要将先序遍历的数组中删除根元素，然后重复上面的过程处理左右两棵子树。还有前序遍历中 左子树序列的值 都在 右子树序列的值前面，所以递归中处理左子树先，数组++完了，在处理右子树的时候，数组pre_inx也到了右子树了

```java
package com.qiming.algorithm.leetcode;

import java.util.HashMap;

/**
 * 从前序与中序遍历序列构造二叉树
 *
 * 根据一棵树的前序遍历与中序遍历构造二叉树。你可以假设树中没有重复的元素。
 * 前序遍历 preorder = [3,9,20,15,7] 中序遍历 inorder = [9,3,15,20,7]
 *     3
 *    / \
 *   9  20
 *     /  \
 *    15   7
 *
 * 思路：首先，preorder 中的第一个元素一定是树的根，这个根又将 inorder 序列分成了左右两棵子树。现在我们只需要将先序遍历的数组中删除根元素，然后重复上面的过程处理左右两棵子树。
 * 还有前序遍历中 左子树序列的值 都在 右子树序列的值前面，所以递归中处理左子树先，数组++完了，在处理右子树的时候，数组pre_inx也到了右子树了
 *
 *
 */
public class ConstructBinaryTreefromPreorderAndInorderTraversal {

  int pre_idx = 0;
  int[] preorder;
  int[] inorder;
  HashMap<Integer, Integer> idx_map = new HashMap<Integer, Integer>();

  public TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal buildTree(int[] preorder, int[] inorder) {

    this.preorder = preorder;
    this.inorder = inorder;

    //构建hashmap，中序中各个数的位置
    int idx = 0;
    for (int i : inorder) {
      idx_map.put(i, idx++);
    }
    return helper(0, inorder.length);

  }

  public TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal helper(int in_left, int in_right) {
    if (in_left == in_right) {
      return null;
    }

    int root_val = preorder[pre_idx];
    TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal root = new TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal(root_val);

    int index = idx_map.get(root_val);
    pre_idx++;
    root.left = helper(in_left, index);
    root.right = helper(index + 1, in_right);
    return root;

  }

}


class TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal {

  int val;
  TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal left;
  TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal right;
  TreeNodeConstructBinaryTreefromPreorderAndInorderTraversal(int x) { val = x; }

}
```

#### 107.4的幂（LC）

题目：给定一个整数 (32 位有符号整数)，请编写一个函数来判断它是否是 4 的幂次方。

思路：看代码正文

```java
package com.qiming.algorithm.leetcode;

/**
 * 4的幂
 *
 * 给定一个整数 (32 位有符号整数)，请编写一个函数来判断它是否是 4 的幂次方。
 *
 * 思路：如何看一个数是否是2的幂，x > 0 and x & (x - 1) == 0，位运算法，先检查num是否是2的幂，现在的问题是区分 2 的偶数幂（当 xx 是 4 的幂时）和 2 的奇数幂（当 xx 不是 4 的幂时）。
 * 在二进制表示中，这两种情况都只有一位为 1，其余为 0。因此 4 的幂与数字 (101010...10)相与会得到0，(101010...10)用16进制表示是(0xaaaaaaaa)
 * 2

 *
 *
 * 作者：LeetCode
 * 链接：https://leetcode-cn.com/problems/power-of-four/solution/4de-mi-by-leetcode/
 * 来源：力扣（LeetCode）
 * 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
 *
 *
 *
 */
public class PowerOfFour {

  public static void main(String[] args) {
    System.out.println(new PowerOfFour().isPowerOfFour(1073741824));
  }

  public boolean isPowerOfFour(int num) {

//    if (num <= 0) {
//      return false;
//    }
//
//    int b = 1;
//    //这边要是16，不然最后一次就进不来了
//    for (int i = 0; i < 16; i++) {
//      if ((num & 2147483647) == b) {
//        return true;
//      } else {
//        b = b << 2;
//      }
//    }
//
//    return false;

    /**
     * 非O(1)的解法
     */
//    if (num == 0) return false;
//    while (num % 4 == 0) num /= 4;
//    return num == 1;

    /**
     * 位运算解法
     */
    return (num > 0) && ((num & (num - 1)) == 0) && ((num & 0xaaaaaaaa) == 0);

  }

}

```

#### 108.二进制求和（LC）

题目：给定两个二进制字符串，返回他们的和（用二进制表示）。输入为非空字符串且只包含数字 1 和 0。

思路：看代码正文

```java
package com.qiming.algorithm.leetcode;

import java.math.BigInteger;

/**
 * 二进制求和
 *
 * 给定两个二进制字符串，返回他们的和（用二进制表示）。输入为非空字符串且只包含数字 1 和 0。
 *
 * 示例 1: 输入: a = "11", b = "1"  输出: "100"
 *
 * 思路：先转换程10进制求和，再转换程2进制，还有一种就是逐位运算了，XOR 操作得到两个数字无进位相加的结果。进位和两个数字与（&）操作结果左移一位对应。
 * 现在问题被简化为：首先计算两个数字的无进位相加结果和进位，然后计算无进位相加结果与进位之和。同理求和问题又可以转换成上一步，直到进位为 0 结束。
 *
 * 把 a 和 b 转换成整型数字 xx 和 yy，xx 保存结果，yy 保存进位。当进位不为 0：y != 0：计算当前 xx 和 yy 的无进位相加结果：answer = x^y。
 * 计算当前 x 和 y 的进位：carry = (x & y) << 1。完成本次循环，更新 x = answer，y = carry。
 *
 * 但是这两个运算或多或少有点问题，如果01串非常长，那么Integer根本也包不下，BigInteger也可能有解析问题，那就纯位运算呢？？
 *
 *
 *
 */
public class AddBinary {

  public static void main(String[] args) {
      char a = 1;
      char b = 2;
    System.out.println(a+b);
  }

  public String addBinary(String a, String b) {

    //这个集合包有点骚
//    return Integer.toBinaryString(Integer.parseInt(a, 2) + Integer.parseInt(b, 2));
//    BigInteger x = new BigInteger(a, 2);
//    BigInteger y = new BigInteger(b, 2);
//    BigInteger zero = new BigInteger("0", 2);
//    BigInteger carry, answer;
//    while (y.compareTo(zero) != 0) {
//      answer = x.xor(y);
//      carry = x.and(y).shiftLeft(1);
//      x = answer;
//      y = carry;
//    }
//    return x.toString(2);

    int n = a.length(), m = b.length();
    if (n < m) return addBinary(b, a);
    int L = Math.max(n, m);

    StringBuilder sb = new StringBuilder();
    int carry = 0, j = m - 1;
    for(int i = L - 1; i > -1; --i) {
      if (a.charAt(i) == '1') ++carry;
      if (j > -1 && b.charAt(j--) == '1') ++carry;

      if (carry % 2 == 1) sb.append('1');
      else sb.append('0');

      carry /= 2;
    }
    if (carry == 1) sb.append('1');
    sb.reverse();

    return sb.toString();


  }

}

```

#### 109.交替打印FooBar（LC）

题目：指定次数的交替打印FooBar

思路：信号量做一波

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

#### 110.H2O 生成（LC）

题目：现在有两种线程，氢 oxygen 和氧 hydrogen，你的目标是组织这两种线程来产生水分子。存在一个屏障（barrier）使得每个线程必须等候直到一个完整水分子能够被产生出来。题目看正文

思路：信号量的多值acquire()和release()

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

#### 111.数组中出现次数超过一半的数字（OF）

题目：数组中有一个数字出现的次数超过数组长度的一般，请找出这个数字，例如。输入一个长度为9的数组{1,2,3,2,2,2,5,4,2}。由于数字2在数组中出现了5次，超过数组长度的一半，因此输出2

思路：基于partition函数的O(n)算法，因为排序之后，数组中间的数字一定就是那个出现次数超过数组长度一半的数字了，递归去做，另外一个思路看正文

```java
package com.qiming.algorithm;

/**
 * 数组中出现次数超过一半的数字
 *
 * 数组中有一个数字出现的次数超过数组长度的一般，请找出这个数字，例如。输入一个长度为9的数组{1,2,3,2,2,2,5,4,2}。由于数字2在数组中出现了5次，超过数组长度的一半，因此输出2
 *
 *
 * 思路，基于partition函数的O(n)算法，因为排序之后，数组中间的数字一定就是那个出现次数超过数组长度一半的数字了，递归去做
 *
 * 还有个想法就有点吊了，数组中有一个数字出现的次数超过数组长度的一半，也就是说它出现的次数比其他所有数字出现的次数的和还要多，因为我们可以考虑在遍历数组的时候保存两个值
 * 一个是数组中的数字，一个是次数，当我们遍历到下一个数字的时候，如果下一个数字和我们之前保存的数字相同，则次数加1，如果下一个数字和我们之前保存的数字不同，则次数减1。
 * 如果次数为零，我们需要保存下一个数字，并把次数设为1。关键来了，由于数字的次数比其他所有数字出现的次数之和还要多，那么要找的数字肯定是最后一次把次数设为1时对应的数字
 *
 */
public class NumOfMorethanHalf {

  public static void main(String[] args) {

    int a[] = {1,2,3,2,2,2,5,4,2};
    System.out.println(new NumOfMorethanHalf().MoreThanHalfNum(a));

  }

  public int MoreThanHalfNum(int number[]) {

    if (number.length == 0) {
      return 0;
    }

    int len = number.length;
    int middle = len >> 1;
    int start = 0;
    int end = len-1;
    int index = partition(number, start, end);
    while (index != middle) {
      if (index > middle) {
        end = index - 1;
        index = partition(number, start, end);
      } else {
        start = index + 1;
        index = partition(number, start, end);
      }
    }

    return number[middle];
    /**
     * 第二种方法
     */
//    int result = number[0];
//    int times = 1;
//    for (int i = 0; i < number.length; i++) {
//      if (times == 0) {
//        result = number[i];
//        times = 1;
//      } else if (number[i] == result) {
//        times++;
//      } else {
//        times--;
//      }
//    }
//    return result;

  }

  private int partition(int a[], int low, int high) {

    int pivot = a[low];
    while (low < high) {
      while (low < high && a[high] >= pivot) {
        high--;
      }
      a[low] = a[high];
      while (low < high && a[low] <= pivot) {
        low++;
      }
      a[high] = a[low];
    }
    a[low] = pivot;
    return low;
  }

}

```
#### 112.最小的K个数（OF）

题目：输入整数数组 arr ，找出其中最小的 k 个数。例如，输入4、5、1、6、2、7、3、8这8个数字，则最小的4个数字是1、2、3、4。

思路：这要看这个数组可否修改，如果可以修改，就可以借鉴partition的思路（排序不算答案了。。），比第k个数字小的数字放到数组的左边，比第k个数字大的数字放到数组的右边，那么数组中左边的k个数字就是最小的k个数字

```java
package com.qiming.algorithm;

/**
 * 最小的K个数
 *
 * 输入整数数组 arr ，找出其中最小的 k 个数。例如，输入4、5、1、6、2、7、3、8这8个数字，则最小的4个数字是1、2、3、4。
 *
 * 示例 1：输入：arr = [3,2,1], k = 2  输出：[1,2] 或者 [2,1]
 * 示例 2：输入：arr = [0,1,2,1], k = 1 输出：[0]
 *
 * 思路：这要看这个数组可否修改，如果可以修改，就可以借鉴partition的思路（排序不算答案了。。），比第k个数字小的数字放到数组的左边，比第k个数字大的数字放到数组的右边，那么数组中
 * 左边的k个数字就是最小的k个数字
 *
 * 还有个O(nlogk)的算法，特别适合处理海量数据。先创建一个大小为k的数据容器来存储最小的k个数字，接下来我们每次从输入的n个数字中读入一个数，如果已有的容器中少于k个，就直接把这个放入数
 * 放入容器中，如果容器已有k个数，也就是容器满了，只能替换了，在k个数中找最大数，有可能要删除这个最大数，有可能要插入一个新的数字，这些操作在logk中最掉，总的就是nlogk。
 * 这个容器是什么呢，就是二叉树，每次都要找到k个整数的最大数字，很容易想到最大堆，在最大堆中，根结点的值总是大于它的子树中的任意结点的值。于是我们每次可以在O(1)得到已有的k个数字中的
 * 最大值，但需要O(logk)时间完成插入和删除，自己从头开始写一个最大堆需要一定的代码，这在面试短短的几十分钟内很难完成，我们可以采用红黑树来实现我们的容器。。。
 *
 */
public class LeastSmallOfK {

  public int[] getLeastNumbers(int[] arr, int k) {

    if (k == 0) {
      return new int[0];
    }

    int result[] = new int[k];
    int start = 0;
    int end = arr.length - 1;
    int index = partition(arr, start, end);
    while (index != k-1) {
      if (index > k - 1) {
        end = index - 1;
        index = partition(arr, start, end);
      } else {
        start = index + 1;
        index = partition(arr, start, end);
      }
    }

    for (int i = 0; i < result.length; i++) {
      result[i] = arr[i];
    }
    return result;

  }

  private int partition(int a[], int low, int high) {

    int pivot = a[low];
    while (low < high) {
      while (low < high && a[high] >= pivot) {
        high--;
      }
      a[low] = a[high];
      while (low < high && a[low] <= pivot) {
        low++;
      }
      a[high] = a[low];
    }
    a[low] = pivot;
    return low;
  }

}

```

#### 113.1～n整数中1出现的次数（OF）

题目：输入一个整数 n ，求1～n这n个整数的十进制表示中1出现的次数。例如，输入12，1～12这些整数中包含1 的数字有1、10、11和12，1一共出现了5次。示例 1：输入：n = 12 输出：5  示例 2： 输入：n = 13 输出：6

思路：这是一道找规律的hard，没那么容易，看代码，同LC233

```java
package com.qiming.algorithm;

/**
 * 1～n整数中1出现的次数
 *
 * 输入一个整数 n ，求1～n这n个整数的十进制表示中1出现的次数。例如，输入12，1～12这些整数中包含1 的数字有1、10、11和12，1一共出现了5次。
 *
 * 示例 1：输入：n = 12 输出：5  示例 2： 输入：n = 13 输出：6
 *
 * 思路：这是一道找规律的hard，没那么容易，看代码
 *
 */
public class NumberOf1Between1AndN {

  public int countDigitOne(int n) {

    return dfs(n);

  }

  private int dfs(int n) {
    if (n <= 0) {
      return 0;
    }

    String numStr = String.valueOf(n);
    int high = numStr.charAt(0) - '0';
    int pow = (int) Math.pow(10, numStr.length() - 1);
    int last = n - high * pow;

    if (high == 1) {
      // 最高位是1，如1234, 此时pow = 1000,那么结果由以下三部分构成：
      // (1) dfs(pow - 1)代表[0,999]中1的个数;
      // (2) dfs(last)代表234中1出现的个数;
      // (3) last+1代表固定高位1有多少种情况。
      return dfs(pow - 1) + dfs(last) + last + 1;
    } else {
      // 最高位不为1，如2234，那么结果也分成以下三部分构成：
      // (1) pow代表固定高位1，有多少种情况;
      // (2) high * dfs(pow - 1)代表999以内和1999以内低三位1出现的个数;
      // (3) dfs(last)同上。
      return pow + high * dfs(pow - 1) + dfs(last);
    }
  }

}

```

#### 113.把数组排成最小的数（OF）

题目：输入一个正整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。

思路：自建一个比较器，比较String的大小，int型拼接可能有溢出风险

```java
package com.qiming.algorithm;

import java.util.Arrays;
import java.util.Comparator;

/**
 * 把数组排成最小的数
 *
 * 输入一个正整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。
 *
 * 示例 2: 输入: [3,30,34,5,9] 输出: "3033459"
 *
 * 思路：自建一个比较器，比较String的大小，int型拼接可能有溢出风险
 *
 */
public class PrintMInNumber {

  public String minNumber(int[] nums) {

    //得到一个String类型数组，形似nums
    String[] strNumbers = new String[nums.length];
    for(int i = 0; i < nums.length; i++) {
      strNumbers[i] = String.valueOf(nums[i]);
    }
    //排序。（传入一个比较器对象）
    Arrays.sort(strNumbers, new Comparator<String>() {
      @Override
      public int compare(String o1, String o2) {
        return (o1 + o2).compareTo(o2 + o1);//升序，交换参数就是降序
      }
    });
    //将该字符串数组元素拼接起来
    StringBuilder sb = new StringBuilder();
    for(int i = 0; i < strNumbers.length; i++) {
      sb.append(strNumbers[i]);
    }
    return sb.toString();
  }

}

```
#### 114.丑数（OF）

题目：我们把只包含因子 2、3 和 5 的数称作丑数（Ugly Number）。求按从小到大的顺序的第 n 个丑数。示例: 输入: n = 10  输出: 12 解释: 1, 2, 3, 4, 5, 6, 8, 9, 10, 12 是前 10 个丑数。 说明:  1 是丑数。 n 不超过1690。

思路：写一个判断丑数的方法，再每个元素比较，但是常规都不是套路，不要在非丑数身上浪费时间，丑数是另一个丑数乘以2、3或者5的结果。因此我们可以创建一个数组，里面的数字是排序好的丑数。这个思路关键在于怎么确保数组里是排序好的。同LC264

```java
package com.qiming.algorithm;

/**
 * 丑数
 *
 * 我们把只包含因子 2、3 和 5 的数称作丑数（Ugly Number）。求按从小到大的顺序的第 n 个丑数。
 *
 * 示例: 输入: n = 10  输出: 12 解释: 1, 2, 3, 4, 5, 6, 8, 9, 10, 12 是前 10 个丑数。 说明:  1 是丑数。 n 不超过1690。
 *
 * 思路：写一个判断丑数的方法，再每个元素比较，但是常规都不是套路
 *
 * 不要在非丑数身上浪费时间，丑数是另一个丑数乘以2、3或者5的结果。因此我们可以创建一个数组，里面的数字是排序好的丑数。这个思路关键在于怎么确保数组里是排序好的
 *
 * 丑数肯定是前面一个数乘以2或3或5的结果，如果现在数组里的最大是M，那么先把所有数乘以2，得到的序列肯定有比M大的，记第一个比M大的数为M2。然后3或5是同样的，记为M3
 * 和M5，所以下一个丑数是M2，M3，M5的较小者，但都乘又不科学，因为已有的丑数是按序排列的，对乘2而言，肯定存在某一个丑数T2，排在它走之前的每一个丑数乘以2得到的结果
 * 都会小于已有最大的丑数，在它之后的结果都会太大，我们只需要记下这个丑数的位置，同时每次生成新的丑数的时候，去更新这个T2，T3和T5一样
 *
 */
public class UglyNumber {

  public int nthUglyNumber(int n) {

    /**
     * 正常遍历的解法
     */
//    if (n <= 0) {
//      return 0;
//    }
//
//    int number = 0;
//    int found = 0;
//    while (found < n) {
//      number++;
//      if (isUglyNumber(number)) {
//        found++;
//      }
//    }
//
//    return number;

    //注意n不超过1690，用辅助数组的思路，记录排序好的数组
    if (n <= 0) {
      return 0;
    }
    int result[] = new int[n];
    result[0] = 1;
    int nextUglyIndex = 1;
    int pMultiply2 = 0;
    int pMultiply3 = 0;
    int pMultiply5 = 0;

    while (nextUglyIndex < n) {
      //记得是索引值和相应数值的乘
      int min = ThreeNumMin(result[pMultiply2] * 2, result[pMultiply3] * 3, result[pMultiply5] * 5);
      result[nextUglyIndex] = min;
      //
      while (result[pMultiply2] * 2 <= result[nextUglyIndex]) {
        pMultiply2++;
      }
      while (result[pMultiply3] * 3 <= result[nextUglyIndex]) {
        pMultiply3++;
      }
      while (result[pMultiply5] * 5 <= result[nextUglyIndex]) {
        pMultiply5++;
      }
      nextUglyIndex++;
    }

    return result[nextUglyIndex-1];
  }

  private int ThreeNumMin (int num1, int num2, int num3) {
    int minfirst = Math.min(num1, num2);
    return Math.min(minfirst, num3);
  }


  //判断丑数的方法
  private boolean isUglyNumber(int n) {

    while (n % 2 == 0) {
      n = n / 2;
    }
    while (n % 3 == 0) {
      n = n / 3;
    }
    while (n % 5 == 0) {
      n = n / 5;
    }
    return (n == 1) ? true : false;
  }

}

```

#### 115.第一个只出现一次的字符（OF）

题目：在字符串 s 中找出第一个只出现一次的字符。如果没有，返回一个单空格。s = "abaccdeff"  返回 "b"  s = ""  返回 " "  0 <= s 的长度 <= 50000

思路：用linkedhashmap，即记录次数，又记录顺序，也可以用辅助数组，两次扫描

```java
package com.qiming.algorithm;

import java.util.LinkedHashMap;
import java.util.Map.Entry;

/**
 * 第一个只出现一次的字符
 *
 * 在字符串 s 中找出第一个只出现一次的字符。如果没有，返回一个单空格。
 *
 * s = "abaccdeff"  返回 "b"  s = ""  返回 " "  0 <= s 的长度 <= 50000
 *
 * 思路：用linkedhashmap，即记录次数，又记录顺序，也可以用辅助数组，两次扫描
 *
 */
public class FirestNotRepeatingChar {

  public char firstUniqChar(String s) {

    /**
     * 这是linkedhashmap的写法
     */
//    LinkedHashMap<Character, Integer> map = new LinkedHashMap<>();
//    for (int i = 0; i < s.length(); i++) {
//      char tmp = s.charAt(i);
//      //这边是不等于null
//      if (map.get(tmp) != null) {
//        map.put(tmp, map.get(tmp) + 1);
//      } else {
//        map.put(tmp, 1);
//      }
//    }
//
//    for (Character character : map.keySet()) {
//      if (map.get(character) == 1) {
//        return character;
//      }
//    }
//
//    return ' ';

    int index[] = new int[26];
    for (int i = 0; i < s.length(); i++) {
      index[s.charAt(i) - 'a']++;
    }

    for (int i = 0; i < s.length(); i++) {
      if (index[s.charAt(i) - 'a'] == 1) {
        return s.charAt(i);
      }
    }

    return ' ';
  }

}

```

#### 116.数组中的逆序对（OF）

题目：在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。输入一个数组，求出这个数组中的逆序对的总数。

思路：循环方式不可取，这题用到的是归并和递归，hard，直接OF看

```java
package com.qiming.algorithm;

/**
 * 数组中的逆序对
 *
 * 在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。输入一个数组，求出这个数组中的逆序对的总数。
 *
 * 示例 1: 输入: [7,5,6,4] 输出: 5  0 <= 数组长度 <= 50000
 *
 * 思路：循环方式不可取，这题用到的是归并和递归，hard，直接OF看
 *
 */
public class InversePairs {

  public int reversePairs(int[] nums) {

    if (nums.length == 0) {
      return 0;
    }

    int copy[] = new int[nums.length];
    for (int i = 0; i < nums.length; i++) {
      copy[i] = nums[i];
    }

    int count = InversePairsCore(nums, copy, 0, nums.length - 1);

    return count;
  }

  private int InversePairsCore(int nums[], int copy[], int start, int end) {

    if (start == end) {
      copy[start] = nums[start];
      return 0;
    }

    int length = (end - start) / 2;
    int left = InversePairsCore(copy, nums, start, start + length);
    int right = InversePairsCore(copy, nums, start + length + 1, end);

    //i初始化为前半段最后一个数字的下标
    int i = start + length;

    //j初始化为后半段最后一个数字的下标
    int j = end;
    int indexCopy = end;
    int count = 0;
    while (i >= start && j>= start + length + 1) {
      if (nums[i] > nums[j]) {
        copy[indexCopy--] = nums[i--];
        count += j - start - length;
      } else {
        copy[indexCopy--] = nums[j--];
      }
    }

    for (; i >= start ; --i) {
      copy[indexCopy--] = nums[i];
    }

    for (; j >= start + length + 1 ; --j) {
      copy[indexCopy--] = nums[j];
    }

    return left + right + count;

  }

}

```

#### 117.两个链表的第一个公共结点（OF）

题目：输入两个链表，找出它们的第一个公共节点。

思路：蛮力不可取，用两个辅助栈呢，从尾部取出比较直到相同，可行，但是辅助空间又是O(m+n)了，用双指针，先走几步的方法。同LC160，其实有问题

```java
package com.qiming.algorithm;

/**
 * 两个链表的第一个公共结点
 *
 * 输入两个链表，找出它们的第一个公共节点。
 *
 * 思路：蛮力不可取，用两个辅助栈呢，从尾部取出比较直到相同，可行，但是辅助空间又是O(m+n)了，用双指针，先走几步的方法
 *
 */
public class IntersectionOfTwoLinkedLists {

  public static void main(String[] args) {

    /**
     * LC这个跟OF测案例是不一样的，执行时不行的
     */
    ListNodeIntersectionOfTwoLinkedLists test1 = new ListNodeIntersectionOfTwoLinkedLists(2);
    ListNodeIntersectionOfTwoLinkedLists test2 = new ListNodeIntersectionOfTwoLinkedLists(0);
    ListNodeIntersectionOfTwoLinkedLists test3 = new ListNodeIntersectionOfTwoLinkedLists(9);
    ListNodeIntersectionOfTwoLinkedLists test4 = new ListNodeIntersectionOfTwoLinkedLists(1);
    ListNodeIntersectionOfTwoLinkedLists test5 = new ListNodeIntersectionOfTwoLinkedLists(2);
    ListNodeIntersectionOfTwoLinkedLists test6 = new ListNodeIntersectionOfTwoLinkedLists(4);

    test2.next = test3;
    test3.next = test4;
    test4.next = test5;
    test5.next = test6;

    ListNodeIntersectionOfTwoLinkedLists result = new IntersectionOfTwoLinkedLists().getIntersectionNode(test1, test2);

    System.out.println(result.val);

  }

  public ListNodeIntersectionOfTwoLinkedLists getIntersectionNode(ListNodeIntersectionOfTwoLinkedLists headA, ListNodeIntersectionOfTwoLinkedLists headB) {

    int lengthA = getListLength(headA);
    int lengthB = getListLength(headB);

    int nLengthDif = lengthA - lengthB;
    ListNodeIntersectionOfTwoLinkedLists longHead = headA;
    ListNodeIntersectionOfTwoLinkedLists shortHead = headB;

    if (lengthB > lengthA) {
      longHead = headB;
      shortHead = headA;
      nLengthDif = lengthB - lengthA;
    }

    //先在长链表上走几步，再同时在两个链表上遍历
    for (int i = 0; i < nLengthDif; i++) {
      longHead = longHead.next;
    }

    while (longHead != null && shortHead != null && longHead.val != shortHead.val) {
      longHead = longHead.next;
      shortHead = shortHead.next;
    }

    return longHead;


  }

  private int getListLength(ListNodeIntersectionOfTwoLinkedLists head) {

    int length = 0;
    ListNodeIntersectionOfTwoLinkedLists p = head;
    while (p != null) {
      length++;
      p = p.next;
    }

    return length;
  }

}


class ListNodeIntersectionOfTwoLinkedLists {
  int val;
  ListNodeIntersectionOfTwoLinkedLists next;
  ListNodeIntersectionOfTwoLinkedLists(int x) {
    val = x;
    next = null;
  }
}
```

#### 118.在排序数组中查找数字 I（OF）

题目：统计一个数字在排序数组中出现的次数。

思路：O(n)不算这个问题中好的算法。最好的是O(logn)。排序的立马想到就是二分查找，先二分查找到target，再线性搜索的话就是O(n)了，好的方法是二分查找第一个和最后一个位置。写出找到第一个target和找到最后一个target的两个函数。同LC34

```java
package com.qiming.algorithm;

/**
 * 在排序数组中查找数字 I
 *
 * 统计一个数字在排序数组中出现的次数。
 *
 * 示例 1: 输入: nums = [5,7,7,8,8,10], target = 8 输出: 2 示例 2: 输入: nums = [5,7,7,8,8,10], target = 6 输出: 0
 *
 * 思路：O(n)不算这个问题中好的算法。最好的是O(logn)。排序的立马想到就是二分查找，先二分查找到target，再线性搜索的话就是O(n)了，好的方法是二分查找第一个和最后一个位置
 *
 * 写出找到第一个target和找到最后一个target的两个函数
 *
 */
public class NumOfTargetInSortedArray {

  public static void main(String[] args) {
    int nums[] = {5,7,7,8,8,10};
    System.out.println(new NumOfTargetInSortedArray().search(nums, 8));
  }

  public int search(int[] nums, int target) {

    int number = 0;
    if (nums.length > 0) {
      int first = GetFirstK(nums, target, 0, nums.length - 1);
      int last = GetLastK(nums, target, 0, nums.length - 1);

      if (first > -1 && last > -1) {
        number = last - first + 1;
      }
    }
    return number;
  }

  private int GetFirstK(int nums[], int target, int start, int end) {

    if (start > end) {
      return -1;
    }

    int middleIndex = (start + end) / 2;
    int middleData = nums[middleIndex];

    if (middleData == target) {
      if ((middleIndex > 0 && nums[middleIndex - 1] != target) || middleIndex == 0) {
        return middleIndex;
      } else {
        end = middleIndex - 1;
      }
    } else if (middleData > target) {
      end = middleIndex - 1;
    } else {
      start = middleIndex + 1;
    }

    return GetFirstK(nums, target, start, end);

  }

  private int GetLastK(int nums[], int target, int start, int end) {

    if (start > end) {
      return -1;
    }

    int middleIndex = (start + end) / 2;
    int middleData = nums[middleIndex];

    if (middleData == target) {
      if ((middleIndex < nums.length - 1 && nums[middleIndex + 1] != target) || middleIndex == nums.length - 1) {
        return middleIndex;
      } else {
        start = middleIndex + 1;
      }
    } else if (middleData < target) {
      start = middleIndex + 1;
    } else {
      end = middleIndex - 1;
    }

    return GetLastK(nums, target, start, end);

  }

}

```

#### 119.平衡二叉树（OF）

题目：输入一棵二叉树的根节点，判断该树是不是平衡二叉树。如果某二叉树中任意节点的左右子树的深度相差不超过1，那么它就是一棵平衡二叉树。

思路：遍历所有结点求左右子树的高度差显然是不可取的，但是这里面有重复计算的，每个节点真正只遍历一次。用后续遍历的方法遍历二叉树，在遍历到下一个节点之前我们已经遍历了它的左右子树，只要在遍历每个节点的时候记录它的深度，我们就可以一边遍历，一边判断这个结点是不是平衡的。同LC110

```java
package com.qiming.algorithm;

/**
 * 平衡二叉树
 *
 * 输入一棵二叉树的根节点，判断该树是不是平衡二叉树。如果某二叉树中任意节点的左右子树的深度相差不超过1，那么它就是一棵平衡二叉树。
 *
 * 思路：遍历所有结点求左右子树的高度差显然是不可取的，但是这里面有重复计算的，每个节点真正只遍历一次。用后续遍历的方法遍历二叉树，在遍历到下一个节点之前
 * 我们已经遍历了它的左右子树，只要在遍历每个节点的时候记录它的深度，我们就可以一边遍历，一边判断这个结点是不是平衡的
 *
 */
public class BalancedBinaryTree {

  public boolean isBalanced(TreeNodeBalancedBinaryTree root) {

    //java中没有int的指针传递，用数组替代
    int depth[] = new int[1];
    depth[0] = 0;
    return isBalanced(root, depth);

  }

  private boolean isBalanced(TreeNodeBalancedBinaryTree root, int[] depth) {

    if (root == null) {
      depth[0] = 0;
      return true;
    }
    int left[] = {0}, right[] = {0};
    if (isBalanced(root.left, left) && isBalanced(root.right, right)) {

      int diff = left[0] - right[0];
      if (diff >= -1 && diff <= 1) {
        depth[0] = 1 + (left[0] > right[0] ? left[0] : right[0]);
        return true;
      }

    }

    return false;
  }

}


class TreeNodeBalancedBinaryTree {

  int val;
  TreeNodeBalancedBinaryTree left;
  TreeNodeBalancedBinaryTree right;
  TreeNodeBalancedBinaryTree(int x) { val = x; }

}
```

#### 120.数组中只出现一次的数字（OF）

题目：一个整型数组 nums 里除两个数字之外，其他数字都出现了两次。请写程序找出这两个只出现一次的数字。要求时间复杂度是O(n)，空间复杂度是O(1)。

思路：运用性质，自己的数字异或后结果为0，如果只有一个数字是1次，那么数组所有数字异或完后就剩下这个1次的数字了，那问题是两个，就要把原数组分成两个子数组。怎么分，还是将原数组所有的元素都异或一遍，结果肯定不为0，也就是二进制中有1的位置，至少，找到第一个为1的位置，记为第n位，现在我们就以第n位是不是1为标准，把原数组分成两个子数组，第一个子数组中每个数字的第n位都是1，第二个子数组中每个数字的第n位都是0，那么出现两次的数字肯定被分配到一个子数组？？因为相同必须每位都同

```java
package com.qiming.algorithm;

/**
 * 数组中只出现一次的数字
 *
 * 一个整型数组 nums 里除两个数字之外，其他数字都出现了两次。请写程序找出这两个只出现一次的数字。要求时间复杂度是O(n)，空间复杂度是O(1)。
 *
 * 输入：nums = [1,2,10,4,1,4,3,3]  输出：[2,10] 或 [10,2]  2 <= nums <= 10000
 *
 * 思路：运用性质，自己的数字异或后结果为0，如果只有一个数字是1次，那么数组所有数字异或完后就剩下这个1次的数字了，那问题是两个，就要把原数组分成两个子数组
 * 怎么分，还是将原数组所有的元素都异或一遍，结果肯定不为0，也就是二进制中有1的位置，至少，找到第一个为1的位置，记为第n位，现在我们就以第n位是不是1为标准
 * 把原数组分成两个子数组，第一个子数组中每个数字的第n位都是1，第二个子数组中每个数字的第n位都是0，那么出现两次的数字肯定被分配到一个子数组？？因为相同必须每位都同
 *
 *
 */
public class FindNumAppearOnce {

  public int[] singleNumbers(int[] nums) {

    int result[] = {0, 0};
    int resultExclusiveOR = 0;
    for (int i = 0; i < nums.length; i++) {
      resultExclusiveOR ^= nums[i];
    }
    int index = findFirstOne(resultExclusiveOR);

    for (int i = 0; i < nums.length; i++) {
      //为1的
      if (isBit1(nums[i], index)) {
        result[0] ^= nums[i];
      } else {
        result[1] ^= nums[i];
      }
    }

    return result;
  }

  private int findFirstOne(int num) {

    int index = 0;
    while ((num & 1) == 0) {
      num = num >> 1;
      index++;
    }
    return index;

  }

  private boolean isBit1(int num, int index) {
    num = num >> index;
    return (num & 1) == 1;
  }

}

```

#### 121.和为s的两个数字（OF）

题目：输入一个递增排序的数组和一个数字s，在数组中查找两个数，使得它们的和正好是s。如果有多对数字的和等于s，则输出任意一对即可。

思路：应用好递增的特性，用两个指针指向第一个元素，跟最后有个元素，后面就是条件判断移哪个元素了

```java
package com.qiming.algorithm;

/**
 * 和为s的两个数字
 *
 * 输入一个递增排序的数组和一个数字s，在数组中查找两个数，使得它们的和正好是s。如果有多对数字的和等于s，则输出任意一对即可。
 *
 * 示例 1：输入：nums = [2,7,11,15], target = 9  输出：[2,7] 或者 [7,2]
 *
 * 思路：应用好递增的特性，用两个指针指向第一个元素，跟最后有个元素，后面就是条件判断移哪个元素了
 *
 */
public class FindNumSWithSum {

  public int[] twoSum(int[] nums, int target) {

    int low = 0;
    int high = nums.length - 1;
    int result[] = {-1,-1};
    while (low < high) {

      int sum = nums[low] + nums[high];

      if (sum == target) {
        result[0] = nums[low];
        result[1] = nums[high];
        return result;
      } else if (sum > target) {
        high--;
      } else {
        low++;
      }
    }
    return result;

  }

}

```
#### 122.和为s的连续正数序列（OF）

题目：输入一个正整数 target ，输出所有和为 target 的连续正整数序列（至少含有两个数）。序列内的数字由小到大排列，不同序列按照首个数字从小到大排列。

思路：考虑两个指针low和high，而low是不能超过(1+s)/2的，不然直接超过s了，利用排序的特性增加low，减小high

```java
package com.qiming.algorithm;


import java.util.LinkedList;

/**
 * 和为s的连续正数序列
 *
 * 输入一个正整数 target ，输出所有和为 target 的连续正整数序列（至少含有两个数）。序列内的数字由小到大排列，不同序列按照首个数字从小到大排列。
 *
 * 思路：考虑两个指针low和high，而low是不能超过(1+s)/2的，不然直接超过s了，利用排序的特性增加low，减小high
 *
 */
public class PrintContinuousSequence {

  public int[][] findContinuousSequence(int target) {

    if (target < 3) {
      return new int[0][0];
    }


    LinkedList<int []> list = new LinkedList();

    int low = 1;
    int high = 2;
    int curSum = low + high;
    int middle = (1 + target) / 2;

    while (low < middle) {
      if (curSum == target) {
        int add[] = new int[high-low+1];
        for (int j = 0, k = low; j < add.length; j++, k++) {
          add[j] = k;
        }
        list.add(add);
      }

      //如果是大于了target。要循环挪下low
      while (curSum > target && low < middle) {
        curSum -= low;
        low++;
        //可能会有值相等的，这里也要判断下
        if (curSum == target) {
          int add[] = new int[high-low+1];
          for (int j = 0, k = low; j < add.length; j++, k++) {
            add[j] = k;
          }
          list.add(add);
        }
      }

      //都要做的
      high++;
      curSum += high;
    }

    int result[][] = new int[list.size()][];

    return list.toArray(result);

  }

}

```

#### 123.翻转单词顺序（OF）

题目：输入一个英文句子，翻转句子中单词的顺序，但单词内字符的顺序不变。

思路：两次翻转，先翻转整个句子，再翻转每个单词，细节是处理空格。同LC151

```java
package com.qiming.algorithm;

/**
 * 翻转单词顺序
 *
 * 输入一个英文句子，翻转句子中单词的顺序，但单词内字符的顺序不变。为简单起见，标点符号和普通字母一样处理。例如输入字符串"I am a student. "，则输出"student. a am I"。
 * 输入字符串可以在前面或者后面包含多余的空格，但是反转后的字符不能包括。如果两个单词间有多余的空格，将反转后单词间的空格减少到只含一个。
 *
 * 思路：两次翻转，先翻转整个句子，再翻转每个单词，细节是处理空格
 *
 */
public class ReverseSentence {

  public static void main(String[] args) {

    String s = "a good   example";

  }

  public String reverseWords(String s) {

    String str = s.trim();
    if (str.equals("")){
      return "";
    }
    //中间有多个空格的分法
    String[] strList = reverse(str).split("\\s+");
    StringBuilder sb = new StringBuilder();
    for (int i=0; i<strList.length; i++){
      sb.append(reverse(strList[i]));
      if(i!=strList.length-1){
        sb.append(" ");
      }
    }
    return sb.toString();

  }

  private String reverse(String s){
    StringBuilder sb = new StringBuilder(s);
    int i=0, j=s.length()-1;
    while(i<j){
      char temp = sb.charAt(i);
      sb.setCharAt(i, sb.charAt(j));
      sb.setCharAt(j, temp);
      i++;
      j--;
    }
    return sb.toString();
  }


}

```

#### 124.左旋转字符串（OF）

题目：字符串的左旋转操作是把字符串前面的若干个字符转移到字符串的尾部。请定义一个函数实现字符串左旋转操作的功能。比如，输入字符串"abcdefg"和数字2，该函数将返回左旋转两位得到的结果"cdefgab"。

思路：同样是翻转，但是我们按两个部分翻转，这两个部分由k间隔，然后对整个字符串翻转，一共翻3次

```java
package com.qiming.algorithm;

/**
 * 左旋转字符串
 *
 * 字符串的左旋转操作是把字符串前面的若干个字符转移到字符串的尾部。请定义一个函数实现字符串左旋转操作的功能。比如，输入字符串"abcdefg"和数字2，
 * 该函数将返回左旋转两位得到的结果"cdefgab"。
 *
 * 输入: s = "abcdefg", k = 2 输出: "cdefgab"
 *
 * 思路：同样是翻转，但是我们按两个部分翻转，这两个部分由k间隔，然后对整个字符串翻转，一共翻3次
 *
 */
public class LeftRotateString {

  public String reverseLeftWords(String s, int n) {

    String s1 = reverse(s.substring(0, n));
    String s2 = reverse(s.substring(n, s.length()));
    return reverse(s1 + s2);


  }

  private String reverse(String s){
    StringBuilder sb = new StringBuilder(s);
    int i=0, j=s.length()-1;
    while(i<j){
      char temp = sb.charAt(i);
      sb.setCharAt(i, sb.charAt(j));
      sb.setCharAt(j, temp);
      i++;
      j--;
    }
    return sb.toString();
  }

}

```

#### 125.二叉搜索树的最近公共祖先（OF）

题目：给定一个二叉搜索树, 找到该树中两个指定节点的最近公共祖先。最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”

思路：相似的还有非二叉搜索树的，要充分利用二叉搜索树的性质。从根节点开始遍历树，如果根结点的值比p，q的值都大，那就去左子树中找，反之亦然，第一个介于p，q的值的结点就是求的结点。同LC235

```java
package com.qiming.algorithm;

/**
 * 二叉搜索树的最近公共祖先
 *
 * 给定一个二叉搜索树, 找到该树中两个指定节点的最近公共祖先。最近公共祖先的定义为：“对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，
 * 满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”
 *
 * 思路：相似的还有非二叉搜索树的，要充分利用二叉搜索树的性质。从根节点开始遍历树，如果根结点的值比p，q的值都大，那就去左子树中找，反之亦然，第一个介于p，q的值的结点就是求的结点
 *
 *
 */
public class LowestCommonAncestorOfABinarySearchTree {

  public TreeNodeLowestCommonAncestorOfABinarySearchTree lowestCommonAncestor(TreeNodeLowestCommonAncestorOfABinarySearchTree root, TreeNodeLowestCommonAncestorOfABinarySearchTree p, TreeNodeLowestCommonAncestorOfABinarySearchTree q) {

    /**
     * 以下是递归的写法
     */
//    int parentVal = root.val;
//
//    // Value of p
//    int pVal = p.val;
//
//    // Value of q;
//    int qVal = q.val;
//
//    if (pVal > parentVal && qVal > parentVal) {
//      // If both p and q are greater than parent
//      return lowestCommonAncestor(root.right, p, q);
//    } else if (pVal < parentVal && qVal < parentVal) {
//      // If both p and q are lesser than parent
//      return lowestCommonAncestor(root.left, p, q);
//    } else {
//      // We have found the split point, i.e. the LCA node.
//      return root;
//    }

    /**
     * 以下是非递归的写法
     */
    // Value of p
    int pVal = p.val;

    // Value of q;
    int qVal = q.val;

    // Start from the root node of the tree
    TreeNodeLowestCommonAncestorOfABinarySearchTree node = root;

    // Traverse the tree
    while (node != null) {

      // Value of ancestor/parent node.
      int parentVal = node.val;

      if (pVal > parentVal && qVal > parentVal) {
        // If both p and q are greater than parent
        node = node.right;
      } else if (pVal < parentVal && qVal < parentVal) {
        // If both p and q are lesser than parent
        node = node.left;
      } else {
        // We have found the split point, i.e. the LCA node.
        return node;
      }
    }
    return null;


  }

}

class TreeNodeLowestCommonAncestorOfABinarySearchTree {

  int val;
  TreeNodeLowestCommonAncestorOfABinarySearchTree left;
  TreeNodeLowestCommonAncestorOfABinarySearchTree right;
  TreeNodeLowestCommonAncestorOfABinarySearchTree(int x) { val = x; }

}

```

#### 126.二叉搜索树的第k大节点（OF）

题目：给定一棵二叉搜索树，请找出其中第k大的节点。

思路：二叉搜索树的特性，中序遍历的时候是排序的。但是这里要看清题目，是第K大节点，不是找最小的K，所以，可以用辅助list做，但是更妥的方法是再用二叉搜索树的性质，也就是先遍历右子树，再左子树，也就是从大到小排列了

```java
package com.qiming.algorithm;

import java.util.ArrayList;
import java.util.List;
import java.util.Stack;

/**
 * 二叉搜索树的第k大节点
 *
 * 给定一棵二叉搜索树，请找出其中第k大的节点。
 *
 * 思路：二叉搜索树的特性，中序遍历的时候是排序的。但是这里要看清题目，是第K大节点，不是找最小的K，所以，可以用辅助list做，但是更妥的方法是再用二叉搜索树的性质，
 * 也就是先遍历右子树，再左子树，也就是从大到小排列了
 *
 */
public class KthLargest {


  public int kthLargest(TreeNodeKthLargest root, int k) {

//    if (root == null) {
//      return -1;
//    }
//    TreeNodeKthLargest p = root;
//    Stack<TreeNodeKthLargest> stack =  new Stack();
//    List<TreeNodeKthLargest> result = new ArrayList();
//    while(p != null || !stack.isEmpty()) {
//      while(p != null) {
//        stack.push(p);
//        p = p.left;
//      }
//      if (!stack.isEmpty()) {
//        p = stack.pop();
//        result.add(p);
//        p = p.right;
//      }
//    }
//
//    return result.get(result.size() - k).val;

    if (root == null) {
      return -1;
    }
    TreeNodeKthLargest p = root;
    Stack<TreeNodeKthLargest> stack =  new Stack();
    int num = 0;
    while(p != null || !stack.isEmpty()) {
      while(p != null) {
        stack.push(p);
        p = p.right;
      }
      if (!stack.isEmpty()) {
        p = stack.pop();
        num++;
        if (num == k) {
          return p.val;
        }
        p = p.left;
      }
    }

    return -1;
  }
}


class TreeNodeKthLargest {
  int val;
  TreeNodeKthLargest left;
  TreeNodeKthLargest right;
  TreeNodeKthLargest(int x) { val = x; }
}


```


#### 127.字符串的排列（OF）

题目：输入一个字符串，打印出该字符串中字符的所有排列。你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。

思路：不是经典回溯，或者OF中的递归，注意这里是要去重的。如"aab"

```java
package com.qiming.algorithm;

import java.util.Arrays;
import java.util.LinkedList;
import java.util.List;

/**
 * 字符串的排列
 *
 * 输入一个字符串，打印出该字符串中字符的所有排列。你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。
 *
 * 示例: 输入：s = "abc" 输出：["abc","acb","bac","bca","cab","cba"] 1 <= s 的长度 <= 8
 *
 * 思路：不是经典回溯，或者OF中的递归，注意这里是要去重的。如"aab"
 *
 */
public class StringPermutation {

  //注意这里在LC不要用static变量
  private List<String> res = new LinkedList<>();

  public static void main(String[] args) {
    String s = "aab";
    new StringPermutation().permutation(s);
  }

  public String[] permutation(String s) {

    LinkedList<Character> track = new LinkedList<Character>();
    char[] chars = s.toCharArray();
    //为啥要sort呢，一定要sort的，跟后面的条件有关
    Arrays.sort(chars);
    backtrack(chars, track, new boolean[s.length()]);
    String[] result = new String[res.size()];
    for (int i = 0; i < res.size(); i++) {
      result[i] = res.get(i);
    }
    return result;
  }

  private void backtrack(char[] chars, LinkedList<Character> track, boolean[] visited) {
    // 触发结束条件
    if (track.size() == chars.length) {
      StringBuilder sb = new StringBuilder();
      for (Character character : track) {
        sb.append(character);
      }
      res.add(sb.toString());
      return;
    }

    for (int i = 0; i < chars.length; i++) {
      // 在递归时，如果当前字符等于上一次的字符并且上一次的索引已经访问过，那么本次遍历跳过
      if (i != 0 && chars[i] == chars[i-1] && visited[i-1])
        continue;
      if (!visited[i]) {
        track.add(chars[i]);
        visited[i] = true;
        // 进入下一层决策树
        backtrack(chars, track, visited);
        // 取消选择
        track.removeLast();
        visited[i] = false;
      }
    }
  }

}

```
#### 128.二叉搜索树与双向链表（OF）

题目：输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的循环双向链表。要求不能创建任何新的节点，只能调整树中节点指针的指向。我们希望将这个二叉搜索树转化为双向循环链表。链表中的每个节点都有一个前驱和后继指针。对于双向循环链表，第一个节点的前驱是最后一个节点，最后一个节点的后继是第一个节点。（对原题有所改动）

思路：中序遍历，认为到根结点的时候，左边是已经排序好的了，而右边在之后排序

```java
package com.qiming.algorithm;

/**
 * 二叉搜索树与双向链表
 *
 * 输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的循环双向链表。要求不能创建任何新的节点，只能调整树中节点指针的指向。我们希望将这个二叉搜索树转化为双向循环链表。
 * 链表中的每个节点都有一个前驱和后继指针。对于双向循环链表，第一个节点的前驱是最后一个节点，最后一个节点的后继是第一个节点。（对原题有所改动）
 *
 *
 * 思路：中序遍历，认为到根结点的时候，左边是已经排序好的了，而右边在之后排序
 *
 *
 */
public class ConvertNode {

  // 中序遍历，访问该节点的时候，对其做如下操作：
  // 1.将当前被访问节点curr的左孩子置为前驱pre（中序）
  // 2.若前驱pre不为空，则前驱的右孩子置为当前被访问节点curr
  // 3.将前驱pre指向当前节点curr，即访问完毕

  // 上述形成的是一个非循环的双向链表
  // 需进行头尾相接
  NodeConvertNode pre=null;

  public NodeConvertNode treeToDoublyList(NodeConvertNode root) {

    if(root==null) return root;
    NodeConvertNode p=root,q=root;
    //下面是要做循环链表
    while(p.left!=null) p=p.left;
    while(q.right!=null) q=q.right;
    inorder(root);
    p.left=q;
    q.right=p;
    return p;


  }

  private void inorder(NodeConvertNode curr){
    if(curr==null) return;

    /**
     * 要用left和right表示前驱和后驱
     */
    inorder(curr.left);

    curr.left=this.pre;
    if(this.pre!=null) this.pre.right=curr;
    pre = curr;

    inorder(curr.right);
  }


}

class NodeConvertNode {
  public int val;
  public NodeConvertNode left;
  public NodeConvertNode right;

  public NodeConvertNode() {}

  public NodeConvertNode(int _val) {
    val = _val;
  }

  public NodeConvertNode(int _val,NodeConvertNode _left,NodeConvertNode _right) {
    val = _val;
    left = _left;
    right = _right;
  }
}
```

#### 129.复杂链表的复制（OF）

题目：请实现 copyRandomList 函数，复制一个复杂链表。在复杂链表中，每个节点除了有一个 next 指针指向下一个节点，还有一个 random 指针指向链表中的任意节点或者 null。

思路：正常的先复制next再复制random需要O(n^2)，如果用辅助空间记录random的位置，则可以缩减到O(n)，如果不用辅助空间，就将复制的链表挨个结点的放到原链表后面，那么random其实就是原random位置的下一个位置了，再遍历得到之后的链表

```java
package com.qiming.algorithm;

/**
 * 复杂链表的复制
 *
 * 请实现 copyRandomList 函数，复制一个复杂链表。在复杂链表中，每个节点除了有一个 next 指针指向下一个节点，还有一个 random 指针指向链表中的任意节点或者 null。
 *
 * 思路：正常的先复制next再复制random需要O(n^2)，如果用辅助空间记录random的位置，则可以缩减到O(n)，如果不用辅助空间，就将复制的链表挨个结点的放到原链表后面，那么random
 * 其实就是原random位置的下一个位置了，再遍历得到之后的链表
 *
 */
public class CloneNodes {

  public static void main(String[] args) {



  }

  public NodeCloneNodes copyRandomList(NodeCloneNodes head) {

    if (head == null) return null;
    NodeCloneNodes p = head;
    while (p!=null) {
      NodeCloneNodes cur = p;
      p = p.next;
      NodeCloneNodes copy = copy(cur);
      cur.next = copy;
    }
    p = head;
    for (int i=0;p!=null;i++,p=p.next) {
      // 复制的节点
      if ((i&1) == 1) {
        if (p.random!=null) {
          p.random = p.random.next;
        }
      }
    }
    p = head;
    NodeCloneNodes newHead = p.next;
    //这边是断开了两组链表
    while (p!=null) {
      NodeCloneNodes cur = p;
      p = p.next;
      if (cur.next!=null) {
        cur.next = cur.next.next;
      }

    }
    return newHead;

  }

  private NodeCloneNodes copy(NodeCloneNodes node) {
    NodeCloneNodes newNode = new NodeCloneNodes(node.val);
    newNode.next = node.next;
    newNode.random = node.random;
    return newNode;
  }


}


class NodeCloneNodes {
  int val;
  NodeCloneNodes next;
  NodeCloneNodes random;

  public NodeCloneNodes(int val) {
    this.val = val;
    this.next = null;
    this.random = null;
  }
}
```

#### 130.二叉树中和为某一值的路径（OF）

题目：输入一棵二叉树和一个整数，打印出二叉树中节点值的和为输入整数的所有路径。从树的根节点开始往下一直到叶节点所经过的节点形成一条路径。

思路：递归的前序遍历，记得返回上一个节点的时候removeLast，记得拷贝结果

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.List;

/**
 * 二叉树中和为某一值的路径
 *
 * 输入一棵二叉树和一个整数，打印出二叉树中节点值的和为输入整数的所有路径。从树的根节点开始往下一直到叶节点所经过的节点形成一条路径。
 *
 * 思路：递归的前序遍历，记得返回上一个节点的时候removeLast，记得拷贝结果
 *
 */
public class PathSumII {

  List<List<Integer>> list = new LinkedList<>();
  LinkedList<Integer> inner = new LinkedList<>();

  public List<List<Integer>> pathSum(TreeNodePathSumII root, int sum) {

    if (root == null) {
      return list;
    }
    sum -= root.val;
    inner.add(root.val);
    if (root.left == null && root.right == null) {
      if (sum == 0) {
        list.add(new LinkedList<>(inner));  //要拷贝的，跟回溯时一样
      }
    }
    if (root.left != null) {
      pathSum(root.left, sum);
    }
    if (root.right != null) {
      pathSum(root.right, sum);
    }
    inner.removeLast();

    return list;
  }

}

class TreeNodePathSumII {
  int val;
  TreeNodePathSumII left;
  TreeNodePathSumII right;
  TreeNodePathSumII(int x) { val = x; }
}
```

#### 131.二叉搜索树的后序遍历序列（OF）

题目：输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 true，否则返回 false。假设输入的数组的任意两个数字都互不相同。

思路：最右边的值肯定是根结点了，那么前面的元素，有一部分是左子树的，值小于根结点，有一部分是右子树，值大于根结点，递归的看都是这样，不能违背二叉搜索树的定义

```java
package com.qiming.algorithm;

/**
 * 二叉搜索树的后序遍历序列
 *
 * 输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 true，否则返回 false。假设输入的数组的任意两个数字都互不相同。
 *
 * 思路：最右边的值肯定是根结点了，那么前面的元素，有一部分是左子树的，值小于根结点，有一部分是右子树，值大于根结点，递归的看都是这样，不能违背二叉搜索树的定义
 *
 */
public class VerifyPostOrder {

  public boolean verifyPostorder(int[] postorder) {

    return helper(postorder, 0, postorder.length-1);

  }

  private boolean helper(int nums[], int start, int end) {
    if (nums.length == 0) {
      return true;
    }

    int root = nums[end];

    int i = start;
    for (; i < end; i++) {
      if (nums[i] > root) {
        break;
      }
    }

    int j = i;
    for (; j < end; j++) {
      if (nums[j] < root) {
        return false;
      }
    }

    boolean leftTree = true;
    if (i > start) {
      leftTree = helper(nums, start, i - 1);
    }
    boolean rightTree = true;
    if (i < end - 1) {
      rightTree = helper(nums, i, end - 1);
    }

    return leftTree && rightTree;
  }

}

```

#### 132.从上到下打印二叉树（OF）

题目：从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。

思路：树的层次遍历

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;

/**
 * 从上到下打印二叉树
 *
 * 从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。
 *
 * 思路：树的层次遍历
 *
 */
public class PrintFromTopToButtom {

  public int[] levelOrder(TreeNodePrintFromTopToButtom root) {

    if (root == null) {
      return new int[0];
    }

    List<Integer> list = new LinkedList<>();
    Queue<TreeNodePrintFromTopToButtom> queue = new LinkedList();
    queue.offer(root);
    while (!queue.isEmpty()) {
      int size = queue.size();
      for (int i = 0; i < size; i++) {
        TreeNodePrintFromTopToButtom p = queue.poll();
        list.add(p.val);
        if (p.left != null) {
          queue.offer(p.left);
        }
        if (p.right != null) {
          queue.offer(p.right);
        }
      }
    }
    int[] result = new int[list.size()];
    for (int i = 0; i < result.length; i++) {
      result[i] = list.get(i);
    }
    return result;
  }

}


class TreeNodePrintFromTopToButtom {

  int val;
  TreeNodePrintFromTopToButtom left;
  TreeNodePrintFromTopToButtom right;
  TreeNodePrintFromTopToButtom(int x) {val = x;}

}
```

#### 133.栈的压入、弹出序列（OF）

题目：输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否为该栈的弹出顺序。假设压入栈的所有数字均不相等。例如，序列 {1,2,3,4,5} 是某栈的压栈序列，序列 {4,5,3,2,1} 是该压栈序列对应的一个弹出序列，但 {4,3,5,1,2} 就不可能是该压栈序列的弹出序列。

思路：如果下一个弹出的数字刚好是栈顶的数字，那么就直接弹出，如果下一个弹出的数字不在栈顶，我们就把压栈序列中还没有入栈的数字压入辅助栈，直到把下一个需要弹出的数字压入栈顶为止，如果所有的数字都压入栈了仍然没有找到下一个弹出的数字，那么该序列不可能是一个弹出序列

```java
package com.qiming.algorithm;

import java.util.Stack;

/**
 * 栈的压入、弹出序列
 *
 * 输入两个整数序列，第一个序列表示栈的压入顺序，请判断第二个序列是否为该栈的弹出顺序。假设压入栈的所有数字均不相等。例如，序列 {1,2,3,4,5} 是某栈的压栈序列，
 * 序列 {4,5,3,2,1} 是该压栈序列对应的一个弹出序列，但 {4,3,5,1,2} 就不可能是该压栈序列的弹出序列。
 *
 * 思路：如果下一个弹出的数字刚好是栈顶的数字，那么就直接弹出，如果下一个弹出的数字不在栈顶，我们就把压栈序列中还没有入栈的数字压入辅助栈，直到把下一个需要弹出的数字
 * 压入栈顶为止，如果所有的数字都压入栈了仍然没有找到下一个弹出的数字，那么该序列不可能是一个弹出序列
 *
 *
 */
public class ValidateStackSequences {

  public boolean validateStackSequences(int[] pushed, int[] popped) {

    if (pushed.length == 0 && popped.length == 0) {
      return true;
    }

    Stack<Integer> stack = new Stack();
    int pushIndex = 0;

    for (int poppedIndex = 0; poppedIndex < popped.length; poppedIndex++) {
      //当还有数可以入栈并且 栈顶元素和要弹出的数不一样，那么继续入栈
      while (pushIndex < pushed.length && (stack.empty() || stack.peek() != popped[poppedIndex])) {
        stack.push(pushed[pushIndex++]);
      }
      if (stack.peek() != popped[poppedIndex]) {
        return false;
      } else {
        stack.pop();
      }

    }

    return true;
  }

}

```

#### 134.删除链表的节点O(1)（OF）

题目：给定单向链表的头指针和一个要删除的节点的值，定义一个函数删除该节点。返回删除后的链表的头节点。主要是要求时间复杂度在O(1)

思路：核心思想就是将待删除结点的下一个结点的值赋给待删除结点，再更新next，注意边界一个是尾结点要遍历，一个是只有一个结点要删除

```java
package com.qiming.algorithm;

/**
 * 删除链表的节点O(1)
 *
 * 给定单向链表的头指针和一个要删除的节点的值，定义一个函数删除该节点。返回删除后的链表的头节点。主要是要求时间复杂度在O(1)
 *
 * 思路：核心思想就是将待删除结点的下一个结点的值赋给待删除结点，再更新next，注意边界一个是尾结点要遍历，一个是只有一个结点要删除
 *
 */
public class DeleteNode {

  public ListNodeDeleteNode deleteNode(ListNodeDeleteNode head, ListNodeDeleteNode val) {

    if (head == null || val == null) {
      return null;
    }

    if (val.next != null) {
      //待删除的结点不是尾结点
      ListNodeDeleteNode next = val.next;
      val.val = next.val;
      val.next = next.next;
    } else if (head == val) {
      //待删除的只有一个结点，且就是头结点
      head = null;
    } else {
      //待删除的是尾结点，需要遍历
      ListNodeDeleteNode cur = head;
      while (cur.next != val) {
        cur = cur.next;
      }
      cur.next = null;
    }

    return head;
  }

}



class ListNodeDeleteNode {
  int val;
  ListNodeDeleteNode next;
  ListNodeDeleteNode(int x) { val = x; }
}
```


#### 135.打印从1到最大的n位数（OF）

题目：输入数字 n，按顺序打印出从 1 到最大的 n 位十进制数。比如输入 3，则打印出 1、2、3 一直到最大的 3 位数 999。

思路：其实是要考虑大数的，转成字符做，但LC的返回是int，那就是正常做了

```java
package com.qiming.algorithm;

/**
 * 打印从1到最大的n位数
 *
 * 输入数字 n，按顺序打印出从 1 到最大的 n 位十进制数。比如输入 3，则打印出 1、2、3 一直到最大的 3 位数 999。
 * 示例 1: 输入: n = 1 输出: [1,2,3,4,5,6,7,8,9]
 *
 * 思路：其实是要考虑大数的，转成字符做，但LC的返回是int，那就是正常做了
 *
 */
public class Print1ToMaxOfDigits {

  public int[] printNumbers(int n) {

    //因为n为正整数，所以最小为10，也可以把10改为9，100改为99等
    int[] map = { 10, 100, 1000, 10_000, 100_000, 1_000_000, 10_000_000,
        100_000_000, 1_000_000_000, Integer.MAX_VALUE };
    int size = map[n-1];
    int[] ans = new int[size - 1];
    for (int i = 1; i < size; i++) {
      ans[i - 1] = i;
    }
    return ans;

  }

}

```

#### 136.数值的整数次方（OF）

题目：实现函数double Power(double base, int exponent)，求base的exponent次方。不得使用库函数，同时不需要考虑大数问题。

思路：OF中思路，x^n次方其实可以分奇偶拆成 x^(n/2) * x^(n/2) 和 x^(n-1/2) * x^(n-1/2) * x，这样。然后考虑临界值和负数比正数多。同LC50

```java
package com.qiming.algorithm;

/**
 * 数值的整数次方
 *
 * 实现函数double Power(double base, int exponent)，求base的exponent次方。不得使用库函数，同时不需要考虑大数问题。
 * 示例 1: 输入: 2.00000, 10 输出: 1024.00000  示例 2: 输入: 2.10000, 3 输出: 9.26100
 *
 * -100.0 < x < 100.0  其数值范围是 [−2^31, 2^31 − 1]
 *
 * 思路：OF中思路，x^n次方其实可以分奇偶拆成 x^(n/2) * x^(n/2) 和 x^(n-1/2) * x^(n-1/2) * x，这样。然后考虑临界值和负数比正数多
 *
 */
public class Power {

  public static void main(String[] args) {

    new Power().myPow(2,8);

  }

  public double myPow(double x, int n) {

    if (x == 0 && n <= 0) {
      return 0;
    }

    long count = n;
    boolean isNeg = false;

    //n为负数时取得正可能越界，估扩大范围
    if (count < 0) {
      isNeg = true;
      count = -1 * count;
    }

    double res = 1;

    while (count != 0) {
      if ((count & 1) == 1) {
        //奇数的情况，需要乘一个自己，其余都是倍乘
        res = res * x;
      }
      count >>= 1;
      x *= x;
    }

    return isNeg ? 1 / res : res;
  }


}

```

#### 137.二进制中1的个数（OF）

题目：请实现一个函数，输入一个整数，输出该数二进制表示中 1 的个数。例如，把 9 表示成二进制是 1001，有 2 位是 1。因此，如果输入 9，则该函数输出 2。

思路：发现数值减1的规律，当数值减1的时候，其实就把数的最右边的1变成了0，并且将其后的所有位取反，那么与的结果是那位1到后面的位数都为0，高位不变，这也就找到了一个1。同LC191

```java
package com.qiming.algorithm;

/**
 * 二进制中1的个数
 *
 * 请实现一个函数，输入一个整数，输出该数二进制表示中 1 的个数。例如，把 9 表示成二进制是 1001，有 2 位是 1。因此，如果输入 9，则该函数输出 2。
 *
 * 思路：发现数值减1的规律，当数值减1的时候，其实就把数的最右边的1变成了0，并且将其后的所有位取反，那么与的结果是那位1到后面的位数都为0，高位不变，这也就找到了一个1
 *
 */
public class NumberOf1 {

  public int hammingWeight(int n) {
    int count = 0;
    while (n != 0) {
      count++;
      n = (n-1) & n;
    }
    return count;
  }

}

```

#### 138.旋转数组的最小数字（OF）

题目：把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。输入一个递增排序的数组的一个旋转，输出旋转数组的最小元素。

思路：这个其实是二分查找法的演化，旋转后要充分利用性质，分成了两个子数组，其中第一个的值都要比第二个大，且都排序，第一个指针指向第一个元素，第二个指针指向最后，取中间值进行比较，更改一二指针，但是这边有个特殊情况就是[1,0,1,1,1]和[1,1,1,0,1]都是[0,1,1,1,1]的旋转，但是不知道mid是落在第一区间还是第二区间了，需要顺序查找，同LC154

```java
package com.qiming.algorithm;

/**
 * 旋转数组的最小数字
 *
 * 把一个数组最开始的若干个元素搬到数组的末尾，我们称之为数组的旋转。输入一个递增排序的数组的一个旋转，输出旋转数组的最小元素。
 * 例如，数组 [3,4,5,1,2] 为 [1,2,3,4,5] 的一个旋转，该数组的最小值为1。
 *
 * 思路：这个其实是二分查找法的演化，旋转后要充分利用性质，分成了两个子数组，其中第一个的值都要比第二个大，且都排序，第一个指针指向第一个元素，第二个指针指向最后，
 * 取中间值进行比较，更改一二指针，但是这边有个特殊情况就是[1,0,1,1,1]和[1,1,1,0,1]都是[0,1,1,1,1]的旋转，但是不知道mid是落在第一区间还是第二区间了，需要顺序查找
 *
 */
public class FindMinOfArray {

  public int minArray(int[] numbers) {

    if (numbers.length == 0) {
      return -1;
    }

    int index1 = 0;
    int index2 = numbers.length - 1;
    int indexMid = index1;

    while (numbers[index1] >= numbers[index2]) {

      if (index2 - index1 == 1) {
        indexMid = index2;
        break;
      }

      indexMid = (index1 + index2) / 2;
      //如果下标index1，index2和indexMid都相等，则顺序查找了
      if (numbers[index1] == numbers[index2] && numbers[index2] == numbers[indexMid]) {
        return MinInOrder(numbers, index1, index2);
      }

      if (numbers[indexMid] >= numbers[index1]) {
        index1 = indexMid;
      } else if (numbers[indexMid] <= numbers[index2]) {
        index2 = indexMid;
      }

    }
    return numbers[indexMid];
  }

  private int MinInOrder(int[] nums, int index1, int index2) {
    int result = nums[index1];
    for (int i = index1 + 1; i <= index2 ; i++) {
      if (result > nums[i]) {
        result = nums[i];
      }
    }
    return result;
  }

}

```

#### 139.二维数组中的查找（OF）

题目：在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。

思路：找由上角的数字，根据递增的特性，不断缩写行和列，同LC240

```java
package com.qiming.algorithm;

/**
 * 二维数组中的查找
 *
 * 在一个 n * m 的二维数组中，每一行都按照从左到右递增的顺序排序，每一列都按照从上到下递增的顺序排序。请完成一个函数，输入这样的一个二维数组和一个整数，判断数组中是否含有该整数。
 *
 * 思路：找由上角的数字，根据递增的特性，不断缩写行和列
 *
 */
public class FindNumberIn2DArray {

  public static void main(String[] args) {

  }

  public boolean findNumberIn2DArray(int[][] matrix, int target) {

    boolean found = false;

    if (matrix.length > 0 && matrix[0].length > 0) {
      int row = 0;
      int column = matrix[0].length - 1;
      while (row < matrix.length && column >= 0) {
        if (matrix[row][column] == target) {
          found = true;
          break;
        } else if (matrix[row][column] > target) {
          column--;
        } else {
          row++;
        }
      }
    }

    return found;

  }

}

```

#### 140.青蛙跳台阶问题（OF）

题目：一只青蛙一次可以跳上1级台阶，也可以跳上2级台阶。求该青蛙跳上一个 n 级的台阶总共有多少种跳法。答案需要取模 1e9+7（1000000007），如计算初始结果为：1000000008，请返回 1。

思路：斐波那契数列的变体，想好公式，只是初始化的值不太一样

```java
package com.qiming.algorithm;

/**
 * 青蛙跳台阶问题
 *
 * 一只青蛙一次可以跳上1级台阶，也可以跳上2级台阶。求该青蛙跳上一个 n 级的台阶总共有多少种跳法。
 *
 * 答案需要取模 1e9+7（1000000007），如计算初始结果为：1000000008，请返回 1。
 *
 *
 * 思路：斐波那契数列的变体，想好公式，只是初始化的值不太一样
 *
 */
public class FrogJump {

  public int numWays(int n) {
    if (n == 1) {
      return 1;
    }
    if (n == 0) {
      return 1;
    }
    long fibNMinusOne = 1;
    long fibNMinusTwo = 1;
    long fibN = 0;
    for (int i = 2; i <= n; i++) {
      fibN = (fibNMinusOne + fibNMinusTwo) % 1000000007;
      fibNMinusTwo = fibNMinusOne;
      fibNMinusOne = fibN;
    }
    if (fibN == 1000000008) {
      return 1;
    }
    return (int)fibN;

  }

}

```

#### 141.滑动窗口最大值（OF）

题目：给定一个数组 nums 和滑动窗口的大小 k，请找出所有滑动窗口里的最大值。输入: nums = [1,3,-1,-3,5,3,6,7], 和 k = 3，输出: [3,3,5,5,6,7]

思路：动态规划的有点吊，就是两个数组left和right，反向思维，建立dp表，然后前提是要分块

```java
package com.qiming.algorithm;

/**
 * 滑动窗口最大值
 *
 * 给定一个数组 nums 和滑动窗口的大小 k，请找出所有滑动窗口里的最大值。输入: nums = [1,3,-1,-3,5,3,6,7], 和 k = 3，输出: [3,3,5,5,6,7]
 *
 * 思路：动态规划的有点吊，就是两个数组left和right，反向思维，建立dp表，然后前提是要分块
 *
 */
public class SlidingWindowMaximum {

  public int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    if (n * k == 0) return new int[0];
    if (k == 1) return nums;

    int [] left = new int[n];
    left[0] = nums[0];
    int [] right = new int[n];
    right[n - 1] = nums[n - 1];
    for (int i = 1; i < n; i++) {
      // from left to right
      if (i % k == 0) left[i] = nums[i];  // block_start
      else left[i] = Math.max(left[i - 1], nums[i]);

      // from right to left
      int j = n - i - 1;
      if ((j + 1) % k == 0) right[j] = nums[j];  // block_end
      else right[j] = Math.max(right[j + 1], nums[j]);
    }

    int [] output = new int[n - k + 1];
    for (int i = 0; i < n - k + 1; i++)
      output[i] = Math.max(left[i + k - 1], right[i]);

    return output;

  }

}

```

#### 142.最长不含重复字符的子字符串（OF）

题目：请从字符串中找出一个最长的不包含重复字符的子字符串，计算该最长子字符串的长度。输入: "abcabcbb" 输出: 3 解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。

思路：先不考虑暴力的解法了，第一个是正常的滑动窗口，第二个是优化的滑动窗口，第三个是代替了Map的滑动，都掌握一下，具体看代码里

```java
package com.qiming.algorithm.leetcode;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * 最长不含重复字符的子字符串
 *
 * 请从字符串中找出一个最长的不包含重复字符的子字符串，计算该最长子字符串的长度。
 *
 * 输入: "abcabcbb" 输出: 3 解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
 *
 * 思路：先不考虑暴力的解法了，第一个是正常的滑动窗口，第二个是优化的滑动窗口，第三个是代替了Map的滑动，都掌握一下，具体看代码里
 *
 */
public class LengthOfLongestSubstring {

  public int lengthOfLongestSubstring(String s) {

    /**
     * 正常的滑动窗口
     * 因为hashset中contains方法是O(1)的，所以要用hashset去做滑动窗口
     */
//    int n = s.length();
//    Set<Character> set = new HashSet<>();
//    //向右侧滑动索引 j，如果它不在 HashSet 中，我们会继续滑动 j。直到 s[j] 已经存在于 HashSet 中。此时，我们找到的没有重复字符的最长子字符串将会以索引 i开头。
//    // 如果我们对所有的 i这样做，就可以得到答案。
//    int ans = 0, i = 0, j = 0;
//    while (i < n && j < n) {
//      if (!set.contains(s.charAt(j))) {
//        set.add(s.charAt(j++));
//        ans = Math.max(ans, j - i); //j加完后的
//      } else {
//        set.remove(s.charAt(i++)); //从头开始移除元素
//      }
//    }
//    return ans;

    //这个部够好，最糟糕的情况下每个字符会被i和j访问两次 aaaa的情况

    /**
     * 优化的滑动窗口
     * 上述的步骤最多需要执行2n个步骤，事实上，可以优化成n步骤
     */
    //定义字符到索引的映射，而不是使用集合来判断一个字符是否存在。 当我们找到重复的字符时，我们可以立即跳过该窗口。
//    int n = s.length(), ans = 0;
//    Map<Character, Integer> map = new HashMap();
//    for (int j = 0, i = 0; j < n; j++) {
//      //注意这里的i是最近的一个重复点，而且这个i不是以0开头的
//      if (map.containsKey(s.charAt(j))) {
//        i = Math.max(map.get(s.charAt(j)), i);
//      }
//      ans = Math.max(ans, j - i + 1);
//      //注意这边的put是j+1，所以0对应的是1
//      map.put(s.charAt(j), j + 1);
//    }
//    return ans;

    /**
     * 再优化
     * 在字符集较小的时候，用数组去替代上面的hashmap，也能起到效果
     */
    int n = s.length(), ans = 0;
    int []index = new int[128];
    for (int j = 0, i = 0; j < n; j++) {
      //这里没有判断其实是默认值为0而已
      i = Math.max(index[s.charAt(j)], i);
      ans = Math.max(ans, j-i+1);
      index[s.charAt(j)] = j + 1;
    }
    return ans;
  }

}

```

#### 143.n个骰子的点数（OF）

题目：把n个骰子扔在地上，所有骰子朝上一面的点数之和为s。输入n，打印出s的所有可能的值出现的概率。你需要用一个浮点数数组返回答案，其中第 i 个元素代表这 n 个骰子所能掷出的点数集合中第 i 小的那个的概率。

思路：其实还是动态规划，想出状态转移方程，f(n,s)是n个骰子和为s的排列情况总数，那么其实f(n,s) = f(n-1, s-1) + f(n-1, s-2) + f(n-1, s-3) + f(n-1, s-4) + f(n-1, s-5) + f(n-1, s-6)，因为下一个骰子的情况有6种，初始阶段的解是 n=1，f(1,1)=f(1,2)=f(1,3)=f(1,4)=f(1,5)=f(1,6) = 1

```java
package com.qiming.algorithm;

/**
 * n个骰子的点数
 *
 * 把n个骰子扔在地上，所有骰子朝上一面的点数之和为s。输入n，打印出s的所有可能的值出现的概率。
 *
 * 你需要用一个浮点数数组返回答案，其中第 i 个元素代表这 n 个骰子所能掷出的点数集合中第 i 小的那个的概率。
 *
 * 输入: 1 输出: [0.16667,0.16667,0.16667,0.16667,0.16667,0.16667]
 *
 * 思路：其实还是动态规划，想出状态转移方程，f(n,s)是n个骰子和为s的排列情况总数，那么其实f(n,s) = f(n-1, s-1) + f(n-1, s-2) + f(n-1, s-3) + f(n-1, s-4) + f(n-1, s-5) + f(n-1, s-6)
 * 因为下一个骰子的情况有6种，初始阶段的解是 n=1，f(1,1)=f(1,2)=f(1,3)=f(1,4)=f(1,5)=f(1,6) = 1
 *
 */
public class PrintProbability {

  public static void main(String[] args) {

    double[] res = new PrintProbability().twoSum(2);
    for (int i = 0; i < res.length; i++) {
      System.out.println(res[i]);
    }


  }

  public double[] twoSum(int n) {

    int dp[][] = new int[n+1][6*n+1];
    double[] ans = new double[5*n+1];  //所有的范围是n到6n
    double all = Math.pow(6, n);  //这是所有的可能性
    for (int i = 1; i <= 6; i++) {
      dp[1][i] = 1;
    }

    for (int i = 1; i <= n ; i++) {  //骰子的个数
      for (int j = i; j <= 6*n; j++) { //总和大小
        for (int k = 1; k <= 6; k++) {
          //总和大于等于k值时，需要考虑前面的值，否则不需要
          dp[i][j] += j >= k ? dp[i-1][j-k] : 0;
          if (i==n) {
            ans[j-i] = dp[i][j] / all;
          }
        }
      }
    }
    return ans;
  }

}

```

#### 144.扑克牌中的顺子（OF）

题目：从扑克牌中随机抽5张牌，判断是不是一个顺子，即这5张牌是不是连续的。2～10为数字本身，A为1，J为11，Q为12，K为13，而大、小王为 0 ，可以看成任意数字。A 不能视为 14。示例 1: 输入: [1,2,3,4,5] 输出: True 示例 2: 输入: [0,0,1,2,5] 输出: True  数组长度为 5  数组的数取值为 [0, 13] .

思路：先排序，排完序后统计0的个数，统计相邻数字之间的空缺总数，如果空缺的总数小于或者等于0的个数，那么就是连续的，还有，非0的数字是不能出现重复的，不满足顺子的要求

```java
package com.qiming.algorithm;

import java.util.Arrays;

/**
 * 扑克牌中的顺子
 *
 * 从扑克牌中随机抽5张牌，判断是不是一个顺子，即这5张牌是不是连续的。2～10为数字本身，A为1，J为11，Q为12，K为13，而大、小王为 0 ，可以看成任意数字。A 不能视为 14。
 * 示例 1: 输入: [1,2,3,4,5] 输出: True 示例 2: 输入: [0,0,1,2,5] 输出: True  数组长度为 5  数组的数取值为 [0, 13] .
 *
 * 思路：先排序，排完序后统计0的个数，统计相邻数字之间的空缺总数，如果空缺的总数小于或者等于0的个数，那么就是连续的，还有，非0的数字是不能出现重复的，不满足顺子的要求
 *
 */
public class IsContinuous {

  public boolean isStraight(int[] nums) {
    Arrays.sort(nums);
    int numberOfZero = 0;
    int numberOfGap = 0;

    //统计数组中0的个数
    for (int i = 0; i < nums.length; i++) {
      if (nums[i] == 0) {
        numberOfZero++;
      }
    }

    //统计数组中的间隔数目
    int small = numberOfZero;
    int big = small + 1;
    while (big < nums.length) {
      //两个数相等，有对子，不可能是数字
      if (nums[small] == nums[big]) {
        return false;
      }

      numberOfGap += nums[big] - nums[small] - 1;
      small = big;
      big++;

    }
    return (numberOfGap > numberOfZero) ? false : true;
  }

}

```

#### 145.数组中重复的数字（OF）

题目：找出数组中重复的数字。在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。

思路：这题其实有很多解，也要看看当时的要求

```java
package com.qiming.algorithm;

import java.util.HashSet;
import java.util.Set;

/**
 * 数组中重复的数字
 *
 * 找出数组中重复的数字。在一个长度为 n 的数组 nums 里的所有数字都在 0～n-1 的范围内。数组中某些数字是重复的，但不知道有几个数字重复了，也不知道每个数字重复了几次。请找出数组中任意一个重复的数字。
 *
 * 示例 1： 输入：[2, 3, 1, 0, 2, 5, 3]  输出：2 或 3  限制： 2 <= n <= 100000
 *
 * 思路：这题其实有很多解，也要看看当时的要求
 *
 */
public class findRepeatNumber {

  public int findRepeatNumber(int[] nums) {

    Set<Integer> set = new HashSet();
    for (int num : nums) {
      if (!set.contains(num)) {
        set.add(num);
      } else {
        return num;
      }
    }
    return -1;
  }

}

```

#### 146.矩阵中的路径（OF）

题目：请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。路径可以从矩阵中的任意一格开始，每一步可以在矩阵中向左、右、上、下移动一格。如果一条路径经过了矩阵的某一格，那么该路径不能再次进入该格子。例如，在下面的3×4的矩阵中包含一条字符串“bfce”的路径（路径中的字母用加粗标出）。

思路：回溯的变体做法，具体看代码

```java
package com.qiming.algorithm;

/**
 * 矩阵中的路径
 *
 * 请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。路径可以从矩阵中的任意一格开始，每一步可以在矩阵中向左、右、上、下移动一格。
 * 如果一条路径经过了矩阵的某一格，那么该路径不能再次进入该格子。例如，在下面的3×4的矩阵中包含一条字符串“bfce”的路径（路径中的字母用加粗标出）。
 * [["a","b","c","e"],
 * ["s","f","c","s"],
 * ["a","d","e","e"]]
 * 但矩阵中不包含字符串“abfb”的路径，因为字符串的第一个字符b占据了矩阵中的第一行第二个格子之后，路径不能再次进入这个格子。
 *
 * 输入：board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]], word = "ABCCED" 输出：true
 *
 * 1 <= board.length <= 200 1 <= board[i].length <= 200
 *
 * 思路：这一看就是回溯法的做法哦，回溯的同时需要去减枝（此元素和目标字符不同，此元素应被访问）
 * 递归参数：当前元素矩阵board所在的行列索引i和j，当前目标字符串在word中的索引k
 * 终止条件：
 * 1、返回false，行或列索引越界，当前矩阵元素与目标字符不同，当前矩阵元素已经被访问过，后两者可合并
 * 2、返回true，字符串word已经全部匹配，即k=len(word) - 1
 * 递推工作：
 * 1、标记当前矩阵元素，将board[i][j]值暂存于变量tmp，并修改为字符'/'，代表此元素已被访问过，防止之后搜索时重复访问
 * 2、搜索下一单元格，朝当前元素的上、下、左、右四个方向开启下层递归，使用或连接，并记录结果至res
 * 3、还原当前矩阵元素，将tmp暂存值还原至board[i][j]
 * 回溯返回值：返回res，代表是否搜索到目标字符串
 *
 * 回溯模板的一种变体
 *
 */
public class WordSearch {

  public boolean exist(char[][] board, String word) {

    char[] words = word.toCharArray();
    for(int i = 0; i < board.length; i++) {
      for(int j = 0; j < board[0].length; j++) {
        if(dfs(board, words, i, j, 0)) return true;
      }
    }
    return false;

  }

  private boolean dfs(char[][] board, char[] word, int i, int j, int k) {

    if(i >= board.length || i < 0 || j >= board[0].length || j < 0 || board[i][j] != word[k]) return false;
    if(k == word.length - 1) return true;
    char tmp = board[i][j];
    board[i][j] = '/';
    boolean res = dfs(board, word, i + 1, j, k + 1) || dfs(board, word, i - 1, j, k + 1) ||
        dfs(board, word, i, j + 1, k + 1) || dfs(board, word, i , j - 1, k + 1);
    board[i][j] = tmp;
    return res;

  }

}

```

#### 147.机器人的运动范围（OF）

题目：地上有一个m行n列的方格，从坐标 [0,0] 到坐标 [m-1,n-1] 。一个机器人从坐标 [0, 0] 的格子开始移动，它每次可以向左、右、上、下移动一格（不能移动到方格外），也不能进入行坐标和列坐标的数位之和大于k的格子。例如，当k为18时，机器人能够进入方格 [35, 37] ，因为3+5+3+7=18。但它不能进入方格 [35, 38]，因为3+5+3+8=19。请问该机器人能够到达多少个格子？

思路：变体的回溯法和DFS结合

```java
package com.qiming.algorithm;

/**
 * 机器人的运动范围
 *
 * 地上有一个m行n列的方格，从坐标 [0,0] 到坐标 [m-1,n-1] 。一个机器人从坐标 [0, 0] 的格子开始移动，它每次可以向左、右、上、下移动一格（不能移动到方格外），
 * 也不能进入行坐标和列坐标的数位之和大于k的格子。例如，当k为18时，机器人能够进入方格 [35, 37] ，因为3+5+3+7=18。但它不能进入方格 [35, 38]，因为3+5+3+8=19。
 * 请问该机器人能够到达多少个格子？
 *
 * 输入：m = 2, n = 3, k = 1 输出：3  1 <= n,m <= 100  0 <= k <= 20
 *
 * 思路：变体的回溯法和DFS结合
 *
 */
public class RobotMovingCount {

  private int count = 0;

  public int movingCount(int m, int n, int k) {

    boolean[][] flag = new boolean[m][n];
    for (int i = 0; i < flag.length; i++) {
      for (int j = 0; j < flag[0].length; j++) {
        flag[i][j] = false;
      }
    }
    dfs(0, 0, k, m, n, flag);
    return count;

  }

  private void dfs(int row, int column, int k, int m, int n, boolean[][] flag) {

    if (row < 0 || row >= m || column < 0 || column >= n || !isSuit(row, column, k) || flag[row][column]) {
      return;
    }

    count++;
    flag[row][column] = true;
    dfs(row + 1, column, k, m, n, flag);
    dfs(row - 1, column, k, m, n, flag);
    dfs(row, column + 1, k, m, n, flag);
    dfs(row, column - 1, k, m, n, flag);

    //访问过了就访问过了，不需要撤销，不像之前不能访问访问过的结点，上面已经判断了
//    flag[row][column] = false;
  }

  private boolean isSuit(int row, int column, int k) {

    int rowLeft = 0;
    int columnLeft = 0;
    int bitSum = 0;
    //1, 0 是不是不成立的话用&&
    while (row != 0 || column != 0) {
      rowLeft = row % 10;
      columnLeft = column % 10;
      bitSum += rowLeft + columnLeft;
      row = row / 10;
      column = column / 10;
    }
    if (bitSum <= k) {
      return true;
    } else {
      return false;
    }

  }

  public static void main(String[] args) {

//    new RobotMovingCount().isSuit(1, 0, 18);

//    new RobotMovingCount().movingCount(3, 1, 0);

    new RobotMovingCount().movingCount(3, 2, 17);

  }

}

```

#### 148.剪绳子（OF）

题目：给你一根长度为 n 的绳子，请把绳子剪成整数长度的 m 段（m、n都是整数，n>1并且m>1），每段绳子的长度记为 k[0],k[1]...k[m] 。请问 k[0]*k[1]*...*k[m] 可能的最大乘积是多少？例如，当绳子的长度是8时，我们把它剪成长度分别为2、3、3的三段，此时得到的最大乘积是18。

思路：贪心算法，按规律，分解成3是最好的，因为一个数是 整数部分 a（a = n / x） 和余数部分 bb （ b = n % x）。也就是说数字 n 由 a 个 x 和 1 个 b 相加而成。

```java
package com.qiming.algorithm;

/**
 * 剪绳子
 *
 * 给你一根长度为 n 的绳子，请把绳子剪成整数长度的 m 段（m、n都是整数，n>1并且m>1），每段绳子的长度记为 k[0],k[1]...k[m] 。请问 k[0]*k[1]*...*k[m] 可能的最大乘积是多少？
 * 例如，当绳子的长度是8时，我们把它剪成长度分别为2、3、3的三段，此时得到的最大乘积是18。
 *
 * 输入: 10 输出: 36  解释: 10 = 3 + 3 + 4, 3 × 3 × 4 = 36
 *
 * 思路：贪心算法，按规律，分解成3是最好的，因为一个数是 整数部分 a（a = n / x） 和余数部分 bb （ b = n % x）。也就是说数字 n 由 a 个 x 和 1 个 b 相加而成。
 * 1、当 n <= 3 时，按照贪心规则应直接保留原数字，但由于题目要求必须拆分，因此必须拆出一个 1，即直接返回 n - 1；
 * 2、求 n 除以 3 的整数部分 a 和余数部分 b
 * 3、当 b == 0 时，直接返回 3^a
 * 4、当 b == 1时，要将一个 1 + 3转换为 2+2（因为大），此时返回 3^{a-1} * 4
 * 5、当 b == 2时，返回 3^a * b
 *
 *
 */
public class CuttingRopeI {

  public int integerBreak(int n) {
    if(n <= 3) return n - 1;
    int a = n / 3, b = n % 3;
    if(b == 0) return (int)Math.pow(3, a);
    if(b == 1) return (int)Math.pow(3, a - 1) * 4;
    return (int)Math.pow(3, a) * 2;
  }

}

```

#### 149.对称的二叉树（OF）

题目：请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。

思路：递归解决，还有个是输出一棵树的镜像，两者的递归套路是不一样的哦

```java
package com.qiming.algorithm;

/**
 * 对称的二叉树
 *
 * 请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。
 *
 * 思路：递归解决，还有个是输出一棵树的镜像，两者的递归套路是不一样的哦
 *
 */
public class SymmetricTree {

  public boolean isSymmetric(TreeNodeSymmetricTree root) {

    return isMirror(root, root);

  }

  private boolean isMirror(TreeNodeSymmetricTree t1, TreeNodeSymmetricTree t2) {

    if (t1 == null && t2 == null) {
      return true;
    }
    if (t1 == null || t2 == null) {
      return false;
    }
    return (t1.val == t2.val) && isMirror(t1.right, t2.left) && isMirror(t1.left, t2.right);

  }



}

class TreeNodeSymmetricTree {

  int val;
  TreeNodeSymmetricTree left;
  TreeNodeSymmetricTree right;
  TreeNodeSymmetricTree(int x) { val = x; }

}

```

#### 150.从上到下打印二叉树 II（OF）

题目：从上到下按层打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。

思路：二叉树的层次遍历，只是输出的返回值不一样，调整一下代码即可

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;

/**
 * 从上到下打印二叉树 II
 *
 * 从上到下按层打印二叉树，同一层的节点按从左到右的顺序打印，每一层打印到一行。
 *
 * [
 *   [3],
 *   [9,20],
 *   [15,7]
 * ]
 *
 * 思路：二叉树的层次遍历，只是输出的返回值不一样，调整一下代码即可
 *
 */
public class BinaryTreeLevelOrderTraversalII {

  public List<List<Integer>> levelOrder(TreeNodeBinaryTreeLevelOrderTraversalII root) {

    if (root == null) {
      return new LinkedList<>();
    }

    List<List<Integer>> list = new LinkedList<>();
    Queue<TreeNodeBinaryTreeLevelOrderTraversalII> queue = new LinkedList();
    queue.offer(root);
    while (!queue.isEmpty()) {
      int size = queue.size();
      List<Integer> inside = new LinkedList<>();
      for (int i = 0; i < size; i++) {
        TreeNodeBinaryTreeLevelOrderTraversalII p = queue.poll();
        inside.add(p.val);
        if (p.left != null) {
          queue.offer(p.left);
        }
        if (p.right != null) {
          queue.offer(p.right);
        }
      }
      list.add(inside);
    }

    return list;

  }

}


class TreeNodeBinaryTreeLevelOrderTraversalII{

  int val;
  TreeNodeBinaryTreeLevelOrderTraversalII left;
  TreeNodeBinaryTreeLevelOrderTraversalII right;
  TreeNodeBinaryTreeLevelOrderTraversalII(int x) { val = x; }


}
```

#### 151.从上到下打印二叉树 III（OF）

题目：请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。

思路：还是老套路，但是加个boolean变量，每层变化下方向，配合linkedlist的特性

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.List;
import java.util.Queue;

/**
 * 从上到下打印二叉树 III
 *
 * 请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。
 *
 * 思路：还是老套路，但是加个boolean变量，每层变化下方向，配合linkedlist的特性
 *
 *
 */
public class BinaryTreeLevelOrderTraversalIII {

  public List<List<Integer>> levelOrder(TreeNodeBinaryTreeLevelOrderTraversalIII root) {

    if (root == null) {
      return new LinkedList<>();
    }

    List<List<Integer>> list = new LinkedList<>();
    Queue<TreeNodeBinaryTreeLevelOrderTraversalIII> queue = new LinkedList();
    queue.offer(root);
    boolean leftToRight = true;
    while (!queue.isEmpty()) {
      int size = queue.size();
      LinkedList<Integer> inside = new LinkedList<>();
      for (int i = 0; i < size; i++) {
        TreeNodeBinaryTreeLevelOrderTraversalIII p = queue.poll();
        if (leftToRight) {
          inside.addLast(p.val);
        } else {
          inside.addFirst(p.val);
        }

        if (p.left != null) {
          queue.offer(p.left);
        }
        if (p.right != null) {
          queue.offer(p.right);
        }
      }
      list.add(inside);
      leftToRight = !leftToRight;
    }

    return list;

  }

}


class TreeNodeBinaryTreeLevelOrderTraversalIII {

  int val;
  TreeNodeBinaryTreeLevelOrderTraversalIII left;
  TreeNodeBinaryTreeLevelOrderTraversalIII right;
  TreeNodeBinaryTreeLevelOrderTraversalIII(int x) { val = x; }

}
```

#### 152.二叉树的序列化与反序列化（OF）

题目：序列化是将一个数据结构或者对象转换为连续的比特位的操作，进而可以将转换后的数据存储在一个文件或者内存中，同时也可以通过网络传输到另一个计算机环境，采取相反方式重构得到原数据。请设计一个算法来实现二叉树的序列化与反序列化。这里不限定你的序列 / 反序列化算法执行逻辑，你只需要保证一个二叉树可以被序列化为一个字符串并且将这个字符串反序列化为原始的树结构。

思路：二叉树的DFS或者BFS，然后再反序列化，这边用先序

```java
package com.qiming.algorithm;

import java.util.Arrays;
import java.util.LinkedList;
import java.util.List;

/**
 * 二叉树的序列化与反序列化
 *
 * 序列化是将一个数据结构或者对象转换为连续的比特位的操作，进而可以将转换后的数据存储在一个文件或者内存中，同时也可以通过网络传输到另一个计算机环境，采取相反方式重构得到原数据。
 * 请设计一个算法来实现二叉树的序列化与反序列化。这里不限定你的序列 / 反序列化算法执行逻辑，你只需要保证一个二叉树可以被序列化为一个字符串并且将这个字符串反序列化为原始的树结构。
 * 你可以将以下二叉树：
 *
 *     1
 *    / \
 *   2   3
 *      / \
 *     4   5
 * 序列化为 "[1,2,3,null,null,4,5]"
 *
 * 说明: 不要使用类的成员 / 全局 / 静态变量来存储状态，你的序列化和反序列化算法应该是无状态的。
 *
 * 思路：二叉树的DFS或者BFS，然后再反序列化，这边用先序
 *
 */
public class SerializeAndDeserializeBinaryTree {

  // Encodes a tree to a single string.
  public String serialize(TreeNodeSerializeAndDeserializeBinaryTree root) {
    return rserialize(root, "");
  }

  public String rserialize(TreeNodeSerializeAndDeserializeBinaryTree root, String str) {

    if (root == null) {
      str += "null,";
    } else {
      str += String.valueOf(root.val) + ",";
      str = rserialize(root.left, str);
      str = rserialize(root.right, str);
    }

    return str;
  }

  // Decodes your encoded data to tree.
  public TreeNodeSerializeAndDeserializeBinaryTree deserialize(String data) {

    String dataArray[] = data.split(",");
    List<String> dataList = new LinkedList<String>(Arrays.asList(dataArray));
    return rdeserialize(dataList);

  }

  public TreeNodeSerializeAndDeserializeBinaryTree rdeserialize(List<String> l) {

    if (l.get(0).equals("null")) {
      l.remove(0);
      return null;
    }
    TreeNodeSerializeAndDeserializeBinaryTree root = new TreeNodeSerializeAndDeserializeBinaryTree(Integer.valueOf(l.get(0)));
    l.remove(0);
    //再先序遍历回去
    root.left = rdeserialize(l);
    root.right = rdeserialize(l);

    return root;

  }

}

class TreeNodeSerializeAndDeserializeBinaryTree {

  int val;
  TreeNodeSerializeAndDeserializeBinaryTree left;
  TreeNodeSerializeAndDeserializeBinaryTree right;
  TreeNodeSerializeAndDeserializeBinaryTree(int x) { val = x; }

}

// Your Codec object will be instantiated and called as such:
// Codec codec = new Codec();
// codec.deserialize(codec.serialize(root));
```

#### 153.数字序列中某一位的数字（OF）

题目：数字以123456789101112131415…的格式序列化到一个字符序列中。在这个序列中，第5位（从下标0开始计数）是5，第13位是1，第19位是4，等等。

思路：有规律的，1位的数字是9个 2位的数字是 90个 3位的数字是 900 又分别对应 9位数，180位数 和 2700位数

```java
package com.qiming.algorithm;

/**
 * 数字序列中某一位的数字
 *
 * 数字以123456789101112131415…的格式序列化到一个字符序列中。在这个序列中，第5位（从下标0开始计数）是5，第13位是1，第19位是4，等等。
 *
 * 请写一个函数，求任意第n位对应的数字。
 *
 * 思路：有规律的，1位的数字是9个 2位的数字是 90个 3位的数字是 900 又分别对应 9位数，180位数 和 2700位数
 *
 */
public class FindNthDigit {

  public int findNthDigit(int n) {
    long num=n;

    long size=1;
    long max=9;

    while(n>0){
      //判断不在当前位数内
      if(num-max*size>0){
        num=num-max*size;
        size++;
        max=max*10;
      }else{
        long count=num/size;
        long left=num%size;
        if(left==0){
          return (int) (((long)Math.pow(10, size-1)+count-1)%10);
        }else{
          return (int) (((long)Math.pow(10, size-1)+count)/((long)Math.pow(10, (size-left)))%10);
        }
      }
    }

    return 0;

  }

}

```

#### 154.0～n-1中缺失的数字（OF）

题目：一个长度为n-1的递增排序数组中的所有数字都是唯一的，并且每个数字都在范围0～n-1之内。在范围0～n-1内的n个数字中有且只有一个数字不在该数组中，请找出这个数字。

思路：其实该是用二分的，但是下面这个写的也很快啊

```java
package com.qiming.algorithm;

/**
 * 0～n-1中缺失的数字
 *
 * 一个长度为n-1的递增排序数组中的所有数字都是唯一的，并且每个数字都在范围0～n-1之内。在范围0～n-1内的n个数字中有且只有一个数字不在该数组中，请找出这个数字。
 * 示例 1: 输入: [0,1,3] 输出: 2
 *
 * 思路：其实该是用二分的，但是下面这个写的也很快啊
 *
 */
public class MissingNumber {

  public int missingNumber(int[] nums) {

    int i = 0;
    for (; i < nums.length; i++) {
      if (i != nums[i]) {
        if (i == 0) {
          return nums[i] - 1;
        } else {
          return i;
        }
      }
    }

    return i + 1;
  }

}

```

#### 155.队列的最大值（OF）

题目：请定义一个队列并实现函数 max_value 得到队列里的最大值，要求函数max_value、push_back 和 pop_front 的时间复杂度都是O(1)。若队列为空，pop_front 和 max_value 需要返回 -1

思路：跟栈那个相似，需要用到辅助最大值队列，注意push和pop都需要有操作

```java
package com.qiming.algorithm;

import java.util.ArrayDeque;
import java.util.Deque;
import java.util.Queue;

/**
 * 队列的最大值
 *
 * 请定义一个队列并实现函数 max_value 得到队列里的最大值，要求函数max_value、push_back 和 pop_front 的时间复杂度都是O(1)。
 * 若队列为空，pop_front 和 max_value 需要返回 -1
 *
 * 思路：跟栈那个相似，需要用到辅助最大值队列，注意push和pop都需要有操作
 */
public class MaxQueue {

  private Queue<Integer> queue;
  //辅助队列，由大到小排列的队列，
  private Deque<Integer> maxQueue;

  public MaxQueue() {
    this.queue = new ArrayDeque<>();
    this.maxQueue = new ArrayDeque<>();
  }

  public int max_value() {
    if (maxQueue.isEmpty()) {
      return -1;
    }
    return maxQueue.peek();
  }

  public void push_back(int value) {
    queue.add(value);
    //比它小的就没有意义了，从队尾删除
    while (!maxQueue.isEmpty() && value > maxQueue.getLast()) {
      maxQueue.pollLast();
    }
    maxQueue.add(value);
  }

  public int pop_front() {
    if (queue.isEmpty()) {
      return -1;
    }
    int ans = queue.poll();
    if (ans == maxQueue.peek()) {
      maxQueue.poll();
    }
    return ans;
  }

}

```

#### 156.构建乘积数组（OF）

题目：给定一个数组 A[0,1,…,n-1]，请构建一个数组 B[0,1,…,n-1]，其中 B 中的元素 B[i]=A[0]×A[1]×…×A[i-1]×A[i+1]×…×A[n-1]。不能使用除法。

思路：使用左右数组来进行存储，L[i]存储a[i]左边的乘积，R[i]存储a[i]右边的乘积，空间换时间的做法，最后有三次遍历

```java
package com.qiming.algorithm;

/**
 * 构建乘积数组
 *
 * 给定一个数组 A[0,1,…,n-1]，请构建一个数组 B[0,1,…,n-1]，其中 B 中的元素 B[i]=A[0]×A[1]×…×A[i-1]×A[i+1]×…×A[n-1]。不能使用除法。
 * 示例: 输入: [1,2,3,4,5] 输出: [120,60,40,30,24]
 * 提示：所有元素乘积之和不会溢出 32 位整数 a.length <= 100000
 *
 * 思路：使用左右数组来进行存储，L[i]存储a[i]左边的乘积，R[i]存储a[i]右边的乘积，空间换时间的做法，最后有三次遍历
 *
 */
public class BuildProductArray {

  public int[] constructArr(int[] a) {

    if (a.length == 0) {
      return new int[0];
    }

    int L[] = new int[a.length];
    int R[] = new int[a.length];
    L[0] = 1;
    R[R.length - 1] = 1;

    for (int i = 1; i < L.length; i++) {
      L[i] = L[i-1] * a[i-1];
    }

    for (int i = R.length - 2; i >= 0; i--) {
      R[i] = R[i+1] * a[i+1];
    }

    for (int i = 0; i < L.length; i++) {
      L[i] = L[i] * R[i];
    }

    return L;

  }

}

```

#### 157.把字符串转换成整数（OF）

题目：写一个函数 StrToInt，实现把字符串转换成整数这个功能。不能使用库函数。

思路：正常一步步往下写，需要有各种判断

```java
package com.qiming.algorithm;

/**
 * 把字符串转换成整数
 *
 * 写一个函数 StrToInt，实现把字符串转换成整数这个功能。不能使用库函数。
 * 首先，该函数会根据需要丢弃无用的开头空格字符，直到寻找到第一个非空格的字符为止。当我们寻找到的第一个非空字符为正或者负号时，则将该符号与之后面尽可能多的连续数字组合起来，
 * 作为该整数的正负号；假如第一个非空字符是数字，则直接将其与之后连续的数字字符组合起来，形成整数。该字符串除了有效的整数部分之后也可能会存在多余的字符，这些字符可以被忽略
 * 它们对于函数不应该造成影响。注意：假如该字符串中的第一个非空格字符不是一个有效整数字符、字符串为空或字符串仅包含空白字符时，则你的函数不需要进行转换。在任何情况下，
 * 若函数不能进行有效的转换时，请返回 0。
 *
 * 说明：假设我们的环境只能存储32位大小的有符号整数，那么其数值范围为 [−2^31,  2^31 − 1]。如果数值超过这个范围，请返回 INT_MAX (2^31 − 1) 或 INT_MIN (−2^31) 。
 * 示例 1: 输入: "42" 输出: 42 示例 2: 输入: "   -42" 输出: -42 输入: "4193 with words" 输出: 4193 输入: "words and 987" 输出: 0 输入: "-91283472332" 输出: -2147483648
 *
 * 思路：正常一步步往下写，需要有各种判断
 *
 *
 */
public class StringToInteger {

  public int strToInt(String str) {

    int result = 0;
    char[] chars = str.toCharArray();
    int len = str.length();
    int zf = 1;
    int i = 0;
    int pop = 0;
    for (; i < len; i++) {
      if (chars[i] == ' ') {
        continue;
      } else {

        if (chars[i] == '-') {
          zf = -1;
          i++; //需要i++，因为后面break了，不会跑上面的i++了
          break;
        }

        if (chars[i] == '+') {
          i++; //需要i++，因为后面break了，不会跑上面的i++了
          break;
        }

        if (chars[i] < '0' || chars[i] > '9') {
          return 0;
        } else {
          break;
        }

      }
    }

    if (i == len) {
      return 0;
    }

    //从找到的数开始
    for (; i < len; i++) {

      if (chars[i] < '0' || chars[i] > '9') {
        return result;
      }
      //字符转换成数字的标准做法，减去48
      pop = (chars[i] - 48) * zf;
      //边界条件，要除以10计算，因为pop是新的一位数字了
      if (result > Integer.MAX_VALUE / 10 || (result == Integer.MAX_VALUE / 10 && pop > 7)) {
        return Integer.MAX_VALUE;
      }

      if (result < Integer.MIN_VALUE / 10 || (result == Integer.MIN_VALUE / 10 && pop < -8)) {
        return Integer.MIN_VALUE;
      }

      result = result * 10 + pop;

    }

    return result;

  }

}

```

#### 158.求1+2+…+n（OF）

题目：求 1+2+...+n ，要求不能使用乘除法、for、while、if、else、switch、case等关键字及条件判断语句（A?B:C）。

思路：短路原则。。（就是与或者或条件的优先判断，，这tm叫短路）

```java
package com.qiming.algorithm;

/**
 * 求1+2+…+n
 *
 * 求 1+2+...+n ，要求不能使用乘除法、for、while、if、else、switch、case等关键字及条件判断语句（A?B:C）。
 *
 * 输入: n = 3 输出: 6
 *
 * 思路：短路原则。。（就是与或者或条件的优先判断，，这tm叫短路）
 *
 */
public class SumNums {

  public int sumNums(int n) {

    int result = 0;
    boolean b = n > 0 && (result = n + sumNums(n-1)) > 0;
    return result;

  }

}

```

#### 159.买卖股票的最佳时机（OF）

题目：给定一个数组，它的第 i 个元素是一支给定股票第 i 天的价格。如果你最多只允许完成一笔交易（即买入和卖出一支股票），设计一个算法来计算你所能获取的最大利润。注意你不能在买入股票前卖出股票。

思路：一次遍历的情况，记录max和min去做

```java
package com.qiming.algorithm;

/**
 * 买卖股票的最佳时机
 *
 * 给定一个数组，它的第 i 个元素是一支给定股票第 i 天的价格。如果你最多只允许完成一笔交易（即买入和卖出一支股票），设计一个算法来计算你所能获取的最大利润。
 * 注意你不能在买入股票前卖出股票。
 *
 * 输入: [7,1,5,3,6,4] 输出: 5 在第 2 天（股票价格 = 1）的时候买入，在第 5 天（股票价格 = 6）的时候卖出，最大利润 = 6-1 = 5 。
 *
 * 思路：一次遍历的情况，记录max和min去做
 *
 */
public class BestTimeToBuyAndSellStock {

  public int maxProfit(int[] prices) {

    int minprice = Integer.MAX_VALUE;
    int maxprofit = 0;
    for (int i = 0; i < prices.length; i++) {
      if (prices[i] < minprice)
        minprice = prices[i];
      else if (prices[i] - minprice > maxprofit)
        maxprofit = prices[i] - minprice;
    }
    return maxprofit;

  }

}

```

#### 160.数组中数字出现的次数 II（OF）

题目：在一个数组 nums 中除一个数字只出现一次之外，其他数字都出现了三次。请找出那个只出现一次的数字。

思路：记录每一位不为0的数字出现的次数，如果出现的次数对3取模为1，则说明只出现一次的数字此位也是1，将这些位相加就得到结果了

```java
package com.qiming.algorithm;

/**
 * 数组中数字出现的次数 II
 *
 * 在一个数组 nums 中除一个数字只出现一次之外，其他数字都出现了三次。请找出那个只出现一次的数字。
 * 输入：nums = [3,4,3,3] 输出：4  输入：nums = [9,1,7,9,7,9,7] 输出：1
 *
 *
 * 思路：记录每一位不为0的数字出现的次数，如果出现的次数对3取模为1，则说明只出现一次的数字此位也是1，将这些位相加就得到结果了
 *
 */
public class SingleNumber {

  public static void main(String[] args) {

    int []nums = {3,4,3,3};
    new SingleNumber().singleNumber(nums);

  }

  public int singleNumber(int[] nums) {

    int ans = 0;
    int bit = 1;
    for (int i = 0; i < 32; i++) {
      int count = 0;
      for (int num : nums) {
        //这边是不等于0哦，与上有值就说明位数上是1，不是==1
        if ((num & bit) != 0) {
          count++;
        }
      }
      if (count % 3 == 1) {
        ans += bit;
      }
      bit <<= 1;
    }

    return ans;

  }

}

```

#### 161.礼物的最大价值（OF）

题目：在一个 m*n 的棋盘的每一格都放有一个礼物，每个礼物都有一定的价值（价值大于 0）。你可以从棋盘的左上角开始拿格子里的礼物，并每次向右或者向下移动一格、直到到达棋盘的右下角。给定一个棋盘及其上面的礼物的价值，请计算你最多能拿到多少价值的礼物？

思路：回溯法超时了，还是得要动态规划，dp[i][j]表示到达这个点的最大值，那么dp[i][j] = max(dp[i-1][j],dp[i][j-1]) + grid[i][j]，而初始值是dp[0][0] = grid[0][0] 还有 d[i][0] = d[i-1][0] + grid[i][0]  d[0][j] = d[0][j-1] + grid[0][j]

```java
package com.qiming.algorithm;

import java.util.TreeSet;

/**
 * 礼物的最大价值
 *
 * 在一个 m*n 的棋盘的每一格都放有一个礼物，每个礼物都有一定的价值（价值大于 0）。你可以从棋盘的左上角开始拿格子里的礼物，并每次向右或者向下移动一格、
 * 直到到达棋盘的右下角。给定一个棋盘及其上面的礼物的价值，请计算你最多能拿到多少价值的礼物？
 *
 * 输入:
 * [
 *   [1,3,1],
 *   [1,5,1],
 *   [4,2,1]
 * ]
 *
 * 输出: 12
 *
 * 解释: 路径 1→3→5→2→1 可以拿到最多价值的礼物   0 < grid.length <= 200  0 < grid[0].length <= 200
 *
 * 思路：回溯法超时了，还是得要动态规划，dp[i][j]表示到达这个点的最大值，那么dp[i][j] = max(dp[i-1][j],dp[i][j-1]) + grid[i][j]
 * 而初始值是dp[0][0] = grid[0][0] 还有 d[i][0] = d[i-1][0] + grid[i][0]  d[0][j] = d[0][j-1] + grid[0][j]
 *
 */
public class GiftMaxValue {

  private TreeSet<Integer> result = new TreeSet();
  private int sum;

  public static void main(String[] args) {

    // int test[][] = ({1,3,1},{1,5,1},{4,2,1}); //jekyll
    // int test1[][] = ({1,2},{5,6},{1,1});
    // int result = new GiftMaxValue().maxValue(test);
    // System.out.println(result);

  }

  public int maxValue(int[][] grid) {

//    dfs(0, 0, grid);
//    return result.last();

    int row = grid.length;
    int column = grid[0].length;

    int dp[][] = new int[row][column];

    dp[0][0] = grid[0][0];

    for (int i = 1; i < row; i++) {
      dp[i][0] = dp[i-1][0] + grid[i][0];
    }

    for (int i = 1; i < column; i++) {
      dp[0][i] = dp[0][i-1] + grid[0][i];
    }

    for (int i = 1; i < row; i++) {
      for (int j = 1; j < column; j++) {
        dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]) + grid[i][j];
      }
    }

    return dp[row-1][column-1];

  }


  private void dfs(int row, int column, int[][] grid) {

    if (row >= grid.length || column >= grid[0].length) {
      return;
    }

    if (row == grid.length - 1 && column == grid[0].length - 1) {
      //说明到最后了
      sum += grid[row][column];
      result.add(sum);
      sum -= grid[row][column];
      return;
    }

    sum += grid[row][column];

    dfs(row, column + 1, grid);
    dfs(row + 1, column, grid);

    sum -= grid[row][column];

  }

}

```

#### 162.把数字翻译成字符串（OF）

题目：给定一个数字，我们按照如下规则把它翻译为字符串：0 翻译成 “a” ，1 翻译成 “b”，……，11 翻译成 “l”，……，25 翻译成 “z”。一个数字可能有多个翻译。请编程实现一个函数，用来计算一个数字有多少种不同的翻译方法。

思路：首先，绝大部分树形问题都可以用回溯来解决，这道抽象成树模型后就是求一颗二叉树从根结点到达叶子结点的路径总数。因为每次选择都只有两个，犹如二叉树的分支。对于13，你第一次可以选择一位，就是1，你也可以选择13，所以最多就是两个选择，去走左子树还是右子树，走到叶子结点就返回1，代表这条路径可以到达终点

```java
package com.qiming.algorithm;

/**
 * 把数字翻译成字符串
 *
 * 给定一个数字，我们按照如下规则把它翻译为字符串：0 翻译成 “a” ，1 翻译成 “b”，……，11 翻译成 “l”，……，25 翻译成 “z”。一个数字可能有多个翻译。请编程实现一个函数，
 * 用来计算一个数字有多少种不同的翻译方法。
 *
 * 输入: 12258 输出: 5 解释: 12258有5种不同的翻译，分别是"bccfi", "bwfi", "bczi", "mcfi"和"mzi"
 *
 * 思路：首先，绝大部分树形问题都可以用回溯来解决，这道抽象成树模型后就是求一颗二叉树从根结点到达叶子结点的路径总数。因为每次选择都只有两个，犹如二叉树的分支
 * 对于13，你第一次可以选择一位，就是1，你也可以选择13，所以最多就是两个选择，去走左子树还是右子树，走到叶子结点就返回1，代表这条路径可以到达终点
 *
 *
 *
 */
public class IntegerToString {

  public static void main(String[] args) {

    int num = 542;

//    int result = new IntegerToString().translateNum(num);
//    System.out.println(result);

  }

  public int translateNum(int num) {

    String s = String.valueOf(num);
    return backtrace(s, 0);

  }

  private int backtrace(String s, int index) {
    int n = s.length();

    if (index == n) {
      return 1;
    }
    //特殊情况，看最后怎么判断双位数的不能
    if (index == n -1 || s.charAt(index) == '0' || Integer.valueOf(s.substring(index, index + 2)) > 25) {
      return backtrace(s, index+1);
    }

    return backtrace(s, index + 1) + backtrace(s, index + 2);

  }

}

```

#### 163.数据流中的中位数（OF）

题目：如何得到一个数据流中的中位数？如果从数据流中读出奇数个数值，那么中位数就是所有数值排序之后位于中间的数值。如果从数据流中读出偶数个数值，那么中位数就是所有数值排序之后中间两个数的平均值。

思路：大顶堆和小顶堆，输入的数据分成两部分，较小的一部分lowPart（最大堆）和较大的一部分highPart（最小堆），看java是如何实现的

```java
package com.qiming.algorithm;

import java.util.PriorityQueue;

/**
 * 数据流中的中位数
 *
 * 如何得到一个数据流中的中位数？如果从数据流中读出奇数个数值，那么中位数就是所有数值排序之后位于中间的数值。如果从数据流中读出偶数个数值，
 * 那么中位数就是所有数值排序之后中间两个数的平均值。
 *
 * [2,3,4] 的中位数是 3 [2,3] 的中位数是 (2 + 3) / 2 = 2.5 设计一个支持以下两种操作的数据结构：void addNum(int num) - 从数据流中添加一个整数到数据结构中。
 * double findMedian() - 返回目前所有元素的中位数。
 *
 * 思路：大顶堆和小顶堆，输入的数据分成两部分，较小的一部分lowPart（最大堆）和较大的一部分highPart（最小堆），如果size是奇数，那么中位数就是lowPart的最大值，堆顶
 * 否则最大值就是lowPart和highPart的堆顶的平均值
 *
 * 每进入一个数，先加入lowPart，然后将lowPart的最大值（堆顶）移出到highPart，如果这时size是奇数，此时highPart将最小值移到lowPart
 *
 */
public class FindMedianFromDataStream {

  private PriorityQueue<Integer> lowPart;
  private PriorityQueue<Integer> highPart;
  private int size;

  /** initialize your data structure here. */
  public FindMedianFromDataStream() {
    //y-x，后减前面，是生成最大堆，默认构造是最小堆，这种函数式的写法只是不写Comparator了
    lowPart = new PriorityQueue<>((x, y) -> y -x);
    highPart = new PriorityQueue<>();
    size = 0;
  }

  public void addNum(int num) {

    size++;
    lowPart.offer(num);
    highPart.offer(lowPart.poll());
    if ((size & 1) == 1) {
      lowPart.offer(highPart.poll());
    }

  }

  public double findMedian() {

    if ((size & 1) == 1) {
      return (double)lowPart.peek();
    } else {
      return (double) (lowPart.peek() + highPart.peek()) / 2;
    }

  }

}


/**
 * Your MedianFinder object will be instantiated and called as such:
 * MedianFinder obj = new MedianFinder();
 * obj.addNum(num);
 * double param_2 = obj.findMedian();
 */
```

#### 164.表示数值的字符串（OF）

题目：请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。例如，字符串"+100"、"5e2"、"-123"、"3.1416"、"0123"及"-1E-16"都表示数值，但"12e"、"1a3.14"、"1.2.3"、"+-5"及"12e+5.4"都不是。

思路：本题可以采用《编译原理》里面的确定的有限状态机（DFA）解决。

```java
package com.qiming.algorithm;

/**
 * 表示数值的字符串
 *
 * 请实现一个函数用来判断字符串是否表示数值（包括整数和小数）。例如，字符串"+100"、"5e2"、"-123"、"3.1416"、"0123"及"-1E-16"都表示数值，
 * 但"12e"、"1a3.14"、"1.2.3"、"+-5"及"12e+5.4"都不是。
 *
 *
 * 思路：本题可以采用《编译原理》里面的确定的有限状态机（DFA）解决。构造一个DFA并实现，构造方法可以先写正则表达式，然后转为 DFA，也可以直接写，
 * 我就是直接写的，虽然大概率不会是最简结构（具体请参考《编译器原理》图灵出版社） hard？？
 *
 *
 *
 */
public class ValidNumber {

  public int make(char c) {
    switch(c) {
      case ' ': return 0;
      case '+':
      case '-': return 1;
      case '.': return 3;
      case 'e': return 4;
      default:
        if(c >= 48 && c <= 57) return 2;
    }
    return -1;
  }

  public boolean isNumber(String s) {
    int state = 0;
    int finals = 0b101101000;
    int[][] transfer = new int[][]({ 0, 1, 6, 2,-1},   //jekyll
        {-1,-1, 6, 2,-1},
        {-1,-1, 3,-1,-1},
        { 8,-1, 3,-1, 4},
        {-1, 7, 5,-1,-1},
        { 8,-1, 5,-1,-1},
        { 8,-1, 6, 3, 4},
        {-1,-1, 5,-1,-1},
        { 8,-1,-1,-1,-1});
    char[] ss = s.toCharArray();
    for(int i=0; i < ss.length; ++i) {
      int id = make(ss[i]);
      if (id < 0) return false;
      state = transfer[state][id];
      if (state < 0) return false;
    }
    return (finals & (1 << state)) > 0;
  }


}

```

#### 165.正则表达式匹配（OF）

题目：请实现一个函数用来匹配包含'. '和'*'的正则表达式。模式中的字符'.'表示任意一个字符，而'*'表示它前面的字符可以出现任意次（含0次）。

思路：回溯或者动态规划

```java
package com.qiming.algorithm;

/**
 * 正则表达式匹配
 *
 * 请实现一个函数用来匹配包含'. '和'*'的正则表达式。模式中的字符'.'表示任意一个字符，而'*'表示它前面的字符可以出现任意次（含0次）。
 * 在本题中，匹配是指字符串的所有字符匹配整个模式。例如，字符串"aaa"与模式"a.a"和"ab*ac*a"匹配，但与"aa.a"和"ab*a"均不匹配。
 *
 * 输入: s = "aa" p = "a" 输出: false 解释: "a" 无法匹配 "aa" 整个字符串。
 * 输入: s = "aab" p = "c*a*b" 输出: true 解释: 因为 '*' 表示零个或多个，这里 'c' 为 0 个, 'a' 被重复一次。因此可以匹配字符串 "aab"。
 * 输入: s = "mississippi" p = "mis*is*p*." 输出: false
 *
 * 思路：回溯或者动态规划 hard？？
 *
 */
public class RegularExpressionMatching {

  public boolean isMatch(String s, String p) {

    boolean[][] dp = new boolean[s.length() + 1][p.length() + 1];
    dp[s.length()][p.length()] = true;

    for (int i = s.length(); i >= 0; i--){
      for (int j = p.length() - 1; j >= 0; j--){
        boolean first_match = (i < s.length() &&
            (p.charAt(j) == s.charAt(i) ||
                p.charAt(j) == '.'));
        if (j + 1 < p.length() && p.charAt(j+1) == '*'){
          dp[i][j] = dp[i][j+2] || first_match && dp[i+1][j];
        } else {
          dp[i][j] = first_match && dp[i+1][j+1];
        }
      }
    }
    return dp[0][0];


  }

}

```

#### 166.前K个高频单词（LC）

题目：给一非空的单词列表，返回前 k 个出现次数最多的单词。返回的答案应该按单词出现频率由高到低排序。如果不同的单词有相同出现频率，按字母顺序排序。

思路：hashmap存储单词的次数，然后排序

```java
package com.qiming.algorithm.leetcode;

import java.util.ArrayList;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 前K个高频单词
 *
 * 给一非空的单词列表，返回前 k 个出现次数最多的单词。返回的答案应该按单词出现频率由高到低排序。如果不同的单词有相同出现频率，按字母顺序排序。
 *
 * 思路：hashmap存储单词的次数，然后排序
 *
 *
 */
public class TopKFrequentWords {

  public List<String> topKFrequent(String[] words, int k) {

    Map<String, Integer> count = new HashMap();
    for (String word: words) {
      count.put(word, count.getOrDefault(word, 0) + 1);
    }
    List<String> candidates = new ArrayList(count.keySet());
    Collections.sort(candidates, (w1, w2) -> count.get(w1).equals(count.get(w2)) ?
        w1.compareTo(w2) : count.get(w2) - count.get(w1));

    return candidates.subList(0, k);


  }

}

```

#### 167.合并区间（LC）

题目：给出一个区间的集合，请合并所有重叠的区间。示例 1: 输入: [[1,3],[2,6],[8,10],[15,18]] 输出: [[1,6],[8,10],[15,18]] 解释: 区间 [1,3] 和 [2,6] 重叠, 将它们合并为 [1,6].

思路：如果我们按照区间的 start 大小排序，那么在这个排序的列表中可以合并的区间一定是连续的。首先，我们将列表按上述方式排序。然后，我们将第一个区间插入 merged 数组中，然后按顺序考虑之后的每个区间：如果当前区间的左端点在前一个区间的右端点之后，那么他们不会重合，我们可以直接将这个区间插入 merged 中；否则，他们重合，我们用当前区间的右端点更新前一个区间的右端点 end 如果前者数值比后者大的话。

```java
package com.qiming.algorithm.leetcode;

import java.util.Comparator;
import java.util.LinkedList;

/**
 * 合并区间
 *
 * 给出一个区间的集合，请合并所有重叠的区间。
 *
 * 示例 1: 输入: [[1,3],[2,6],[8,10],[15,18]] 输出: [[1,6],[8,10],[15,18]] 解释: 区间 [1,3] 和 [2,6] 重叠, 将它们合并为 [1,6].
 *
 * 思路：如果我们按照区间的 start 大小排序，那么在这个排序的列表中可以合并的区间一定是连续的。首先，我们将列表按上述方式排序。然后，我们将第一个区间插入 merged 数组中，
 * 然后按顺序考虑之后的每个区间：如果当前区间的左端点在前一个区间的右端点之后，那么他们不会重合，我们可以直接将这个区间插入 merged 中；否则，他们重合，
 * 我们用当前区间的右端点更新前一个区间的右端点 end 如果前者数值比后者大的话。
 *
 *
 */
public class MergeIntervals {
  private static class Interval {
    int start;
    int end;
    Interval(int[] interval) {
      this.start = interval[0];
      this.end = interval[1];
    }

    int[] toArray() {
      return new int[]{this.start, this.end};
    }
  }

  public int[][] merge(int[][] intervals) {

    LinkedList<Interval> intervalsList = new LinkedList<Interval>();
    for (int[] interval : intervals) {
      intervalsList.add(new Interval(interval));
    }

    intervalsList.sort(new IntervalComparator());

    LinkedList<Interval> merged = new LinkedList<Interval>();
    for (Interval interval : intervalsList) {
      // if the list of merged intervals is empty or if the current
      // interval does not overlap with the previous, simply append it.
      if (merged.isEmpty() || merged.getLast().end < interval.start) {
        merged.add(interval);
      }
      // otherwise, there is overlap, so we merge the current and previous
      // intervals.
      else {
        merged.getLast().end = Math.max(merged.getLast().end, interval.end);
      }
    }

    int i = 0;
    int [][]result = new int[merged.size()][2];
    for (Interval interval : merged) {
      result[i] = interval.toArray();
      i++;
    }

    return result;

  }

  private static class IntervalComparator implements Comparator<Interval> {
    @Override
    public int compare(Interval a, Interval b) {
      return Integer.compare(a.start, b.start);
    }
  }


}

```

#### 168.判定字符是否唯一（CI）

题目：实现一个算法，确定一个字符串 s 的所有字符是否全都不同。

思路：不用额外，就是常规的数组，set不要用，那么双循环N^2的时间复杂度也不可取，还有就是位运算了，要熟悉位运算的各种操作

```java
package com.qiming.algorithm.lcci;

/**
 * 判定字符是否唯一
 *
 * 实现一个算法，确定一个字符串 s 的所有字符是否全都不同。
 *
 * 输入: s = "leetcode" 输出: false  输入: s = "abc" 输出: true  0 <= len(s) <= 100，
 * 如果你不使用额外的数据结构，会很加分。
 *
 * 思路：不用额外，就是常规的数组，set不要用，那么双循环N^2的时间复杂度也不可取，还有就是位运算了，要熟悉位运算的各种操作
 *
 */
public class IsStringUnique {

  public boolean isUnique(String astr) {

    long x = 0; //采用位运算来实现空间复杂度O（1）
    int n = astr.length();
    for(int i = 0;i < n; i++){
      int c = astr.charAt(i) - 'a';
      if(((x >> c) & 1) == 1){ //如果这一位是1说明有重复
        return false;
      }
      //其实就是x是一个标志位了，或的操作时肯定能把这一位置为1的
      x |= (1 << c); //把对应位置修改为1
    }
    return true;


  }

}

```

#### 169.判定是否互为字符重排（CI）

题目：给定两个字符串 s1 和 s2，请编写一个程序，确定其中一个字符串的字符重新排列后，能否变成另一个字符串。

思路：用一个辅助数组，然后正常的字符数量解

```java
package com.qiming.algorithm.lcci;

/**
 * 判定是否互为字符重排
 *
 * 给定两个字符串 s1 和 s2，请编写一个程序，确定其中一个字符串的字符重新排列后，能否变成另一个字符串。
 *
 * 输入: s1 = "abc", s2 = "bca" 输出: true  输入: s1 = "abc", s2 = "bad" 输出: false
 *
 * 思路：用一个辅助数组，然后正常的字符数量解
 *
 */
public class CheckPermutation {

  public boolean CheckPermutation(String s1, String s2) {

    int index[] = new int[26];

    for (int i = 0; i < s1.length(); i++) {
      index[s1.charAt(i) - 'a']++;
    }

    for (int i = 0; i < s2.length(); i++) {
      index[s2.charAt(i) - 'a']--;
    }

    for (int i = 0; i < index.length; i++) {
      if (index[i] != 0) {
        return false;
      }
    }

    return true;

  }

}

```

#### 170.回文排列（CI）

题目：给定一个字符串，编写一个函数判定其是否为某个回文串的排列之一。回文串是指正反两个方向都一样的单词或短语。排列是指字母的重新排列。

思路：注意要求，26位的辅助数组是不行的，用256位的，数组也不要减'a'了，奇数个数为0或是1就是回文了

```java
package com.qiming.algorithm.lcci;

/**
 * 回文排列
 *
 * 给定一个字符串，编写一个函数判定其是否为某个回文串的排列之一。回文串是指正反两个方向都一样的单词或短语。排列是指字母的重新排列。
 * 输入："tactcoa" 输出：true（排列有"tacocat"、"atcocta"，等等），示例没说是什么字符，可能出现"AaBb//a"，也就是有大写有特殊字符
 *
 * 思路：注意要求，26位的辅助数组是不行的，用256位的，数组也不要减'a'了，奇数个数为0或是1就是回文了
 *
 */
public class PalindromePermutation {

  public boolean canPermutePalindrome(String s) {

    int index[] = new int[256];
    for (int i = 0; i < s.length(); i++) {
      index[s.charAt(i)]++;
    }

    int oddCount = 0;
    for (int i = 0; i < index.length; i++) {
      if (index[i] % 2 != 0) {
        oddCount++;
      }
    }

    return oddCount <= 1 ? true : false;


  }

}

```

#### 171.一次编辑（CI）

题目：字符串有三种编辑操作:插入一个字符、删除一个字符或者替换一个字符。 给定两个字符串，编写一个函数判定它们是否只需要一次(或者零次)编辑。

思路：正常双指针的思路写的，但是要处理特殊情况，有几个，还是用长短字符去判断比较好，这样省事

```java
package com.qiming.algorithm.lcci;

/**
 * 一次编辑
 *
 * 字符串有三种编辑操作:插入一个字符、删除一个字符或者替换一个字符。 给定两个字符串，编写一个函数判定它们是否只需要一次(或者零次)编辑。
 *
 * 输入: first = "pales" second = "pal" 输出: False
 *
 * 思路：正常双指针的思路写的，但是要处理特殊情况，有几个，还是用长短字符去判断比较好，这样省事
 *
 */
public class OneAway {

  public static void main(String[] args) {

    String first = "ab";
    String second = "bc";

    System.out.println(new OneAway().oneEditAway(first, second));

  }

  public boolean oneEditAway(String first, String second) {

    int i = 0, j =0;
    int gap = 0;
    while (i < first.length() && j < second.length()) {
      if (first.charAt(i) == second.charAt(j)) {
        i++;
        j++;
      } else {
        if ((i + 1) < first.length() && first.charAt(i+1) == second.charAt(j)) {
          gap++;
          i++;
        } else if ((j + 1) < second.length() && first.charAt(i) == second.charAt(j+1)) {
          gap++;
          j++;
        } else if ((i + 1) < first.length() && (j + 1) < second.length() && first.charAt(i+1) == second.charAt(j+1)) {
          gap++;
          i++;
          j++;
        } else {
          gap++;
          i++;
          j++;
        }
      }
      if (gap > 1) {
        return false;
      }
    }

    if ((i != first.length() || j != second.length())) {
      //ab 和 bc的情况
      if (gap >= 1) {
        return false;
      }
      //teacher 和 teach的情况
      if ((first.length() - i) >= 2 || (second.length() - j) >= 2 ) {
        return false;
      }

    }

    return true;

  }

}

```

#### 172.旋转矩阵（CI）

题目：给定一幅由N × N矩阵表示的图像，其中每个像素的大小为4字节，编写一种方法，将图像旋转90度。不占用额外内存空间能否做到？

思路：这个规律其实比较难发现，先进行每个数对角的交换，再进行数组上半和下半的对换，就能实现，找个例子试下

```java
package com.qiming.algorithm.lcci;

/**
 * 旋转矩阵
 *
 * 给定一幅由N × N矩阵表示的图像，其中每个像素的大小为4字节，编写一种方法，将图像旋转90度。不占用额外内存空间能否做到？
 *
 * 思路：这个规律其实比较难发现，先进行每个数对角的交换，再进行数组上半和下半的对换，就能实现，找个例子试下
 *
 */
public class RotateMatrix {

  public void rotate(int[][] matrix) {

    int size = matrix.length;
    for (int i = 0; i < size; i++) {
      for (int j = 0; j < size - i - 1; j++) {
        int swap = matrix[i][j];
        matrix[i][j] = matrix[size - j - 1][size - i - 1];
        matrix[size - j - 1][size - i - 1] = swap;
      }
    }
    for (int i = 0; i < size / 2; i++) {
      for (int j = 0; j < size; j++) {
        int swap = matrix[i][j];
        matrix[i][j] = matrix[size - i - 1][j];
        matrix[size - i - 1][j] = swap;
      }
    }

  }

}

```

#### 173.字符串压缩（CI）

题目：字符串压缩。利用字符重复出现的次数，编写一种方法，实现基本的字符串压缩功能。比如，字符串aabcccccaaa会变为a2b1c5a3。若“压缩”后的字符串没有变短，则返回原先的字符串。你可以假设字符串中只包含大小写英文字母（a至z）

思路：有点滑动窗口的意思，然后有快慢两个指针，注意好边界的判定

```java
package com.qiming.algorithm.lcci;

/**
 * 字符串压缩
 *
 * 字符串压缩。利用字符重复出现的次数，编写一种方法，实现基本的字符串压缩功能。比如，字符串aabcccccaaa会变为a2b1c5a3。若“压缩”后的字符串没有变短，则返回原先的字符串。
 * 你可以假设字符串中只包含大小写英文字母（a至z）
 *
 * 示例1 输入："aabcccccaaa" 输出："a2b1c5a3"  示例2 输入："abbccd" 输出："abbccd" 解释："abbccd"压缩后为"a1b2c2d1"，比原字符串长度更长
 *
 * 思路：有点滑动窗口的意思，然后有快慢两个指针，注意好边界的判定
 *
 */
public class CompressString {

  public static void main(String[] args) {

    String s = "AaBbCcDd";
    String s1 = "aabcccccaaa";
    String s2 = "abbccd";
    System.out.println(new CompressString().compressString(s2));

  }

  public String compressString(String S) {

    int i = 0, j = 1;
    int numCount = 1;
    StringBuilder sb = new StringBuilder();
    while (i < S.length()) {
      if (j < S.length() && S.charAt(i) == S.charAt(j)) {
        j++;
        numCount++;
      } else {
        sb.append(S.charAt(i));
        sb.append(numCount);
        i = j;
        j = i + 1;
        numCount = 1;
      }

    }

    return sb.length() < S.length() ? sb.toString() : S;

  }

}

```

#### 174.零矩阵（CI）

题目：编写一种算法，若M × N矩阵中某个元素为0，则将其所在的行与列清零。

思路：需要设零的行i：将该行的第一列元素matrix[i][0]设为0，表示该行需要清零；需要设为零的列j：将该列的第一行元素matrix[0][j]设为0，表示该列需要清零；特殊处理行列为0

```java
package com.qiming.algorithm.lcci;

/**
 * 零矩阵
 *
 * 编写一种算法，若M × N矩阵中某个元素为0，则将其所在的行与列清零。
 * 输入：
 * [
 *   [1,1,1],
 *   [1,0,1],
 *   [1,1,1]
 * ]
 * 输出：
 * [
 *   [1,0,1],
 *   [0,0,0],
 *   [1,0,1]
 * ]
 *
 * 思路：需要设零的行i：将该行的第一列元素matrix[i][0]设为0，表示该行需要清零；需要设为零的列j：将该列的第一行元素matrix[0][j]设为0，表示该列需要清零；
 * 特殊处理行列为0
 *
 */
public class ZeroMatrix {

  public void setZeroes(int[][] matrix) {

    boolean shu = false;
    boolean hen = false;
    for (int i = 0; i < matrix.length; i++) {
      for (int j = 0; j < matrix[0].length; j++) {
        if (matrix[i][j] == 0) {
          if (i == 0) {
            hen = true;
          }
          if (j == 0) {
            shu = true;
          }
          matrix[i][0] = 0;
          matrix[0][j] = 0;
        }
      }
    }

    for (int i = 1; i < matrix.length; i++) {
      if (matrix[i][0] == 0) {
        for (int j = 1; j < matrix[0].length; j++) {
          matrix[i][j] = 0;
        }
      }
    }

    for (int i = 1; i < matrix[0].length; i++) {
      if (matrix[0][i] == 0) {
        for (int j = 1; j < matrix.length; j++) {
          matrix[j][i] = 0;
        }
      }
    }

    if (shu) {
      for (int i = 0; i < matrix.length; i++) {
        matrix[i][0] = 0;
      }
    }
    if (hen) {
      for (int i = 0; i < matrix[0].length; i++) {
        matrix[0][i] = 0;
      }
    }

  }

}

```
