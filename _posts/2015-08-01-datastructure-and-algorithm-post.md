---
layout: post
title: 数据结构与算法
tags: [code, algorithm, datastructure, java]
author-id: zqmalyssa
---

数据结构与算法，啥也不说了

reference:《数据结构与算法JAVA版》，《剑指offer》

#### 线性表

线性表可以用数组去实现，也可以用链表去实现，统称为线性表


#### 栈

栈可以用数组去实现，数组的末尾当做是栈顶，O(1)

也可以用链表去实现，链表的头部当做是栈顶，O(1)

应用：进制转换，括号匹配检测，迷宫求解

#### 队列

循环数组实现的称为循环队列，用循环队列(front，rear)可以在O(1)时间完成enqueue和dequeue，循环队列的判空可以是用标志位或者多用一个存储单元来实现，因为原始的，空和满的判定条件都是front==rear

也可以用链式存储来实现队列


#### 递归

#### 树

树跟上面不一样，是非线性结构

满二叉树和完全二叉树可以用顺序存储结构(数组)，不是这两种类型的树就要补充虚拟节点

链式存储的话，数据域，左孩子，右孩子

**这边说下二叉树的深度遍历和广度遍历(类比图的DFS和BFS)**

1、DFS，深度优先遍历其实就是不撞南墙不回头的思想，一个结点一个结点的往下寻觅，直到没有后继结点再返回，二叉树有三种DFS，先序，中序，后序遍历，有递归和非递归的版本
	- 深度优先遍历，多使用栈这种数据结构
2、BFS，广度优先遍历其实是将相邻结点都遍历完了再开始下一层，在二叉树中对应的是层次遍历
	- 广度优先遍历，多使用队列这种数据结构


### 图

图是一种较线性结构和树结构更复杂的结构，个人理解是多对多(之前是一对一和一对多)

图的存储方式一个是邻接矩阵，无向图的邻接矩阵是一个对称矩阵，邻接矩阵占用的空间应该是N*N

图的另一种存储方式是邻接表，即图的链式存储方式，而在有向图中，为了方便求得有向图中顶点的入度

优化图的邻接和逆邻接，给出一种双链式的存储结构，顶点还是一条链，边又是一条链


### 查找

顺序查找

如果是排序的，用折半查找

hash表查找

### 排序

排成有序的，方便使用折半查找

排序后前后位置没有发生变化，则是稳定的，排序后前后位置发生变化，则是不稳定的

按类型分为

1. 插入排序
	- 直接插入排序
	- 折半插入排序
	- 希尔排序
2. 交换排序
	- 起泡排序
	- 快速排序
3. 选择排序
	- 简单选择排序
	- 树型选择排序
	- 堆排序
4. 归并排序
	- 分治解决方法，基于合并操作，合并两个有序的序列是容易的，不论是顺序存储还是链式存储，合并操作可以在O(m+n)内完成
	- 划分，将待排序的序列划分为大小相等（或大致相等）的两个子序列
	- 治理，当子序列的规模大于1时，递归排序子序列，如果子序列的规模为1则成为有序序列
	- 组合，将两个有序的子序列合并为一个有序序列

### 动态规划

动态规划套路详解，动态规划算法似乎是一种很高深莫测的算法，你会在一些面试或算法书籍的高级技巧部分看到相关内容，什么状态转移方程，重叠子问题，最优子结构等高大上的词汇也可能让你望而却步。

而且，当你去看用动态规划解决某个问题的代码时，你会觉得这样解决问题竟然如此巧妙，但却难以理解，你可能惊讶于人家是怎么想到这种解法的。

实际上，动态规划是一种常见的「算法设计技巧」，并没有什么高深莫测，至于各种高大上的术语，那是吓唬别人用的，只要你亲自体验几把，这些名词的含义其实显而易见，再简单不过了。

至于为什么最终的解法看起来如此精妙，是因为动态规划遵循一套固定的流程：递归的暴力解法 -> 带备忘录的递归解法 -> 非递归的动态规划解法。这个过程是层层递进的解决问题的过程，你如果没有前面的铺垫，直接看最终的非递归动态规划解法，当然会觉得牛逼而不可及了。

首先，第一个快被举烂了的例子，斐波那契数列，知道递归，备忘录递归和动态规划的演进

第二个，找零问题

题目：给你 k 种面值的硬币，面值分别为 c1, c2 ... ck，再给一个总金额 n，问你最少需要几枚硬币凑出这个金额，如果不可能凑出，则回答 -1 。

比如说，k = 3，面值分别为 1，2，5，总金额 n = 11，那么最少需要 3 枚硬币，即 11 = 5 + 5 + 1 。下面走流程。首先是最困难的一步，写出状态转移方程，这个问题比较好写：

![findmoney]({{ "/assets/img/algorithm/findmoney.png" | relative_url}})

其实，这个方程就用到了「最优子结构」性质：原问题的解由子问题的最优解构成。即 f(11) 由 f(10), f(9), f(6) 的最优解转移而来。这边要理解，很重要

记住，要符合「最优子结构」，子问题间必须互相独立。啥叫相互独立？你肯定不想看数学证明，我用一个直观的例子来讲解。比如说，你的原问题是考出最高的总成绩，那么你的子问题就是要把语文考到最高，数学考到最高...... 为了每门课考到最高，你要把每门课相应的选择题分数拿到最高，填空题分数拿到最高...... 当然，最终就是你每门课都是满分，这就是最高的总成绩。得到了正确的结果：最高的总成绩就是总分。因为这个过程符合最优子结构，“每门科目考到最高”这些子问题是互相独立，互不干扰的。但是，如果加一个条件：你的语文成绩和数学成绩会互相制约，此消彼长。这样的话，显然你能考到的最高总成绩就达不到总分了，按刚才那个思路就会得到错误的结果。因为子问题并不独立，语文数学成绩无法同时最优，所以最优子结构被破坏。回到凑零钱问题，显然子问题之间没有相互制约，而是互相独立的。所以这个状态转移方程是可以得到正确答案的。之后就没啥难点了，按照方程写暴力递归算法即可。

```java
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
```
时间复杂度分析：子问题总数 x 每个子问题的时间。子问题总数为递归树节点个数，这个比较难看出来，是 O(n^k)，总之是指数级别的。每个子问题中含有一个 for 循环，复杂度为 O(k)。所以总时间复杂度为 O(k*n^k)，指数级别。

接着，使用带备忘录的递归算法

```java
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
		//这边相当于使用了备忘录
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
```
很显然「备忘录」大大减小了子问题数目，完全消除了子问题的冗余，所以子问题总数不会超过金额数 n，即子问题数目为 O(n)。处理一个子问题的时间不变，仍是 O(k)，所以总的时间复杂度是 O(kn)。

使用动态规划解决

```java
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

```

![findmoneydp]({{ "/assets/img/algorithm/findmoneydp.png" | relative_url}})

计算机解决问题其实没有任何奇技淫巧，它唯一的解决办法就是穷举，穷举所有可能性。算法设计无非就是先思考“如何穷举”，然后再追求“如何聪明地穷举”。

列出动态转移方程，就是在解决“如何穷举”的问题。之所以说它难，一是因为很多穷举需要递归实现，二是因为有的问题本身的解空间复杂，不那么容易穷举完整。

备忘录、DP table 就是在追求“如何聪明地穷举”。用空间换时间的思路，是降低时间复杂度的不二法门，除此之外，试问，还能玩出啥花活？

### 回溯算法

废话不多说，直接上回溯算法框架。解决一个回溯问题，实际上就是一个决策树的遍历过程。你只需要思考 3 个问题：

1、路径：也就是已经做出的选择。
2、选择列表：也就是你当前可以做的选择。
3、结束条件：也就是到达决策树底层，无法再做选择的条件。

代码方面，回溯算法的框架是

```java
result = []
def backtrack(路径, 选择列表):
    if 满足结束条件:
        result.add(路径)
        return

    for 选择 in 选择列表:
        做选择
        backtrack(路径, 选择列表)
        撤销选择

```
其核心就是 for 循环里面的递归，在递归调用之前「做选择」，在递归调用之后「撤销选择」，特别简单。

通过[全排列]简单解释下

我们在高中的时候就做过排列组合的数学题，我们也知道 n 个不重复的数，全排列共有 n! 个。我们来画一个回溯树

![quanpailie1]({{ "/assets/img/algorithm/quanpailie1.png" | relative_url}})

只要从根遍历这棵树，记录路径上的数字，其实就是所有的全排列。我们不妨把这棵树称为回溯算法的「决策树」，比如中间往下走，你现在就在做决策，可以选择 1 那条树枝，也可以选择 3 那条树枝。为啥只能在 1 和 3 之中选择呢？因为 2 这个树枝在你身后，这个选择你之前做过了，而全排列是不允许重复使用数字的。

现在可以解答开头的几个名词：[2] 就是「路径」，记录你已经做过的选择；[1,3] 就是「选择列表」，表示你当前可以做出的选择；「结束条件」就是遍历到树的底层，在这里就是选择列表为空的时候。

可以把「路径」和「选择」列表作为决策树上每个节点的属性，比如下图列出了几个节点的属性：

![quanpailie2]({{ "/assets/img/algorithm/quanpailie2.png" | relative_url}})

我们定义的 backtrack 函数其实就像一个指针，在这棵树上游走，同时要正确维护每个节点的属性，每当走到树的底层，其「路径」就是一个全排列。

再进一步，如何遍历一棵树？这个应该不难吧。各种搜索问题其实都是树的遍历问题，而多叉树的遍历框架就是这样：

```java
void traverse(TreeNode root) {
    for (TreeNode child : root.childern)
        // 前序遍历需要的操作
        traverse(child);
        // 后序遍历需要的操作
}

```
而所谓的前序遍历和后序遍历，他们只是两个很有用的时间点，我给你画张图你就明白了：

![quanpailie3]({{ "/assets/img/algorithm/quanpailie3.png" | relative_url}})

前序遍历的代码在进入某一个节点之前的那个时间点执行，后序遍历代码在离开某个节点之后的那个时间点执行。

回想我们刚才说的，「路径」和「选择」是每个节点的属性，函数在树上游走要正确维护节点的属性，那么就要在这两个特殊时间点搞点动作：

![quanpailie4]({{ "/assets/img/algorithm/quanpailie4.png" | relative_url}})

核心代码框架就是

```java
for 选择 in 选择列表:
    # 做选择
    将该选择从选择列表移除
    路径.add(选择)
    backtrack(路径, 选择列表)
    # 撤销选择
    路径.remove(选择)
    将该选择再加入选择列表

```
我们只要在递归之前做出选择，在递归之后撤销刚才的选择，就能正确得到每个节点的选择列表和路径。全排列的代码

```java
package com.qiming.algorithm;

import java.util.LinkedList;
import java.util.List;

/**
 * 全排列的代码
 *
 * 使用到了回溯法
 */
public class Permutation {

  private static List<List<Integer>> res = new LinkedList<>();

  public static void main(String[] args) {
    int[] nums = {1, 2, 3, 4};
    permute(nums);
    for (List<Integer> re : res) {
      for (Integer integer : re) {
        System.out.print(integer + " ");
      }
      System.out.println("\n");
    }
  }

  private static List<List<Integer>> permute(int nums[]) {
    //记录[路径]
    LinkedList<Integer> track = new LinkedList<>();
    backtrack(nums, track);
    return res;
  }

  /**
   * 核心的递归，
   * 路径：记录在 track 中
   * 选择列表：nums 中不存在于 track 的那些元素
   * 结束条件：nums 中的元素全都在 track 中出现
   * @param nums
   * @param track
   */
  private static void backtrack(int[] nums, LinkedList<Integer> track) {
    // 触发结束条件
    if (track.size() == nums.length) {
      //这种方法是进行拷贝吧？
      res.add(new LinkedList(track));
      return;
    }

    for (int i = 0; i < nums.length; i++) {
      // 排除不合法的选择
      if (track.contains(nums[i]))
        continue;
      // 做选择
      track.add(nums[i]);
      // 进入下一层决策树
      backtrack(nums, track);
      // 取消选择
      track.removeLast();
    }
  }

}

```
我们这里稍微做了些变通，没有显式记录「选择列表」，而是通过 nums 和 track 推导出当前的选择列表：

但是必须说明的是，不管怎么优化，都符合回溯框架，而且时间复杂度都不可能低于 O(N!)，因为穷举整棵决策树是无法避免的。这也是回溯算法的一个特点，不像动态规划存在重叠子问题可以优化，回溯算法就是纯暴力穷举，复杂度一般都很高。


### Trie树

介绍了以下内容：数据结构 Trie（前缀树）及其最常见的操作。Trie (发音为 "try") 或前缀树是一种树数据结构，用于检索字符串数据集中的键。这一高效的数据结构有多种应用

1. 自动补全
2. 拼写检查
3. IP路由（最长前缀匹配）
4. T9（九宫格）打字预测
5. 单词游戏

还有其他的数据结构，如平衡树和哈希表，使我们能够在字符串数据集中搜索单词。为什么我们还需要 Trie 树呢？尽管哈希表可以在 O(1)O(1) 时间内寻找键值，却无法高效的完成以下操作：

1. 找到具有同一前缀的全部键值。
2. 按词典序枚举字符串的数据集。

Trie 树优于哈希表的另一个理由是，随着哈希表大小增加，会出现大量的冲突，时间复杂度可能增加到 O(n)，其中 n 是插入的键的数量。与哈希表相比，Trie 树在存储多个具有相同前缀的键时可以使用较少的空间。此时 Trie 树只需要 O(m)的时间复杂度，其中 m 为键长。而在平衡树中查找键值需要O(mlogn) 时间复杂度

Trie 树是一个有根的树，其结点具有以下字段：

1. 最多 R个指向子结点的链接，其中每个链接对应字母表数据集中的一个字母。
本文中假定 R为 26，小写拉丁字母的数量。
2. 布尔字段，以指定节点是对应键的结尾还是只是键前缀。

```java
class TrieNode {

    // R links to node children
    private TrieNode[] links;

    private final int R = 26;

    private boolean isEnd;

    public TrieNode() {
        links = new TrieNode[R];
    }

    public boolean containsKey(char ch) {
        return links[ch -'a'] != null;
    }
    public TrieNode get(char ch) {
        return links[ch -'a'];
    }
    public void put(char ch, TrieNode node) {
        links[ch -'a'] = node;
    }
    public void setEnd() {
        isEnd = true;
    }
    public boolean isEnd() {
        return isEnd;
    }
}

```
Trie 树中最常见的两个操作是键的插入和查找

**向 Trie 树中插入键**

我们通过搜索 Trie 树来插入一个键。我们从根开始搜索它对应于第一个键字符的链接。有两种情况

1. 链接存在。沿着链接移动到树的下一个子层。算法继续搜索下一个键字符。
2. 链接不存在。创建一个新的节点，并将它与父节点的链接相连，该链接与当前的键字符相匹配

重复以上步骤，直到到达键的最后一个字符，然后将当前节点标记为结束节点，算法完成。

![trie1]({{ "/assets/img/algorithm/trie1.png" | relative_url}})

```java
class Trie {
    private TrieNode root;

    public Trie() {
        root = new TrieNode();
    }

    // Inserts a word into the trie.
    public void insert(String word) {
        TrieNode node = root;
        for (int i = 0; i < word.length(); i++) {
            char currentChar = word.charAt(i);
            if (!node.containsKey(currentChar)) {
                node.put(currentChar, new TrieNode());
            }
            node = node.get(currentChar);
        }
        node.setEnd();
    }
}

```
时间复杂度：O(m)，其中 m 为键长。在算法的每次迭代中，我们要么检查要么创建一个节点，直到到达键尾。只需要 m 次操作。
空间复杂度：O(m)。最坏的情况下，新插入的键和 Trie 树中已有的键没有公共前缀。此时需要添加 m 个结点，使用 O(m) 空间。

**在 Trie 树中查找键**

每个键在 trie 中表示为从根到内部节点或叶的路径。我们用第一个键字符从根开始，。检查当前节点中与键字符对应的链接。有两种情况：

1. 存在链接。我们移动到该链接后面路径中的下一个节点，并继续搜索下一个键字符。
2. 不存在链接。若已无键字符，且当前结点标记为 isEnd，则返回 true。否则有两种可能，均返回 false :
	- 还有键字符剩余，但无法跟随 Trie 树的键路径，找不到键。
	- 没有键字符剩余，但当前结点没有标记为 isEnd。也就是说，待查找键只是Trie树中另一个键的前缀。

![trie2]({{ "/assets/img/algorithm/trie2.png" | relative_url}})

```java
// search a prefix or whole key in trie and
    // returns the node where search ends
    private TrieNode searchPrefix(String word) {
        TrieNode node = root;
        for (int i = 0; i < word.length(); i++) {
           char curLetter = word.charAt(i);
           if (node.containsKey(curLetter)) {
               node = node.get(curLetter);
           } else {
               return null;
           }
        }
        return node;
    }

    // Returns if the word is in the trie.
    public boolean search(String word) {
       TrieNode node = searchPrefix(word);
       return node != null && node.isEnd();
    }

```
时间复杂度 : O(m)。算法的每一步均搜索下一个键字符。最坏的情况下需要 m 次操作。
空间复杂度 : O(1)。

**查找 Trie 树中的键前缀**

该方法与在 Trie 树中搜索键时使用的方法非常相似。我们从根遍历 Trie 树，直到键前缀中没有字符，或者无法用当前的键字符继续 Trie 中的路径。与上面提到的“搜索键”算法唯一的区别是，到达键前缀的末尾时，总是返回 true。我们不需要考虑当前 Trie 节点是否用 “isend” 标记，因为我们搜索的是键的前缀，而不是整个键。

![trie3]({{ "/assets/img/algorithm/trie3.png" | relative_url}})

```java
// Returns if there is any word in the trie
    // that starts with the given prefix.
    public boolean startsWith(String prefix) {
        TrieNode node = searchPrefix(prefix);
        return node != null;
    }

```
时间复杂度O(m)，空间复杂度O(1)

### 欧几里得算法gcd

欧几里得算法gcd(辗转相除法)， 又名欧几里德算法（Euclidean algorithm），是求最大公约数的一种方法。它的具体做法是：用较小数除较大数，再用出现的余数（第一余数）去除除数，再用出现的余数（第二余数）去除第一余数，如此反复，直到最后余数是0为止。如果是求两个数的最大公约数，那么最后的除数就是这两个数的最大公约数。

a=q*b+r;  都为整数  gcd(a,b)=gcd(b,r);

gcd(a,b)=gcd(b, a mod b );

123456 和 7890 的最大公因数是 6，这可由下列步骤（其中，“a mod b”是指取 a ÷ b 的余数）看出：

| a | b | a mod b |
| :----: | :----:  | :----: |
| 123456 | 7890 | 5106 |
| 7890 | 5106 | 2784 |
| 5106 | 2784 | 2322 |
| 2784 | 2322 | 462 |
| 2322 | 462 | 12 |
| 462 | 12 | 6 |
| 12 | 6 | 0 |

```java
//x,y不分谁大谁小的
public int gcd(int x, int y) {
  return x == 0 ? y : gcd(y%x, x);
}
```

### 贪心算法

在对问题求解时，总是做出在当前看来是最好的选择。也就是说，不从整体最优上加以考虑，他所做出的是在某种意义上的局部最优解。

跳跃游戏：

给定一个非负整数数组，你最初位于数组的第一个位置。数组中的每个元素代表你在该位置可以跳跃的最大长度。你的目标是使用最少的跳跃次数到达数组的最后一个位置。

例如：[2,3,1,1,4,2,2,1] 很明显最短路线：2跳到3的位置，再跳到4的位置，然后就可以跳到最后。

算法思路：（{}表示当前位置，()表示能够达到的最远距离）
从第一个数字2开始，能达到橙色1的位置。能达到的最远位置变更。
{2},3,(1),1,4,2,2,1
走到绿色3的位置，这个位置能达到的最远位置变成橙色4的位置，最远位置变更，步数加一。
2,{3},1,1,(4),2,2,1
继续走到绿色1的位置，这个位置能达到的最远位置不如橙色4的位置，最远位置没有变化，步数不变。
2,3,{1},1,(4),2,2,1
继续走到绿色1的位置，这个位置能达到的最远位置不如橙色4的位置，最远位置没有变化，步数不变
2,3,1,{1},(4),2,2,1
继续走到绿色4的位置，这个位置能达到的最远位置为橙色1的位置，最远位置变化，步数加一，且已经到达终点，循环结束。
2,3,1,1,{4},2,2,(1)

```java
package com.qiming.algorithm;

public class JumpGame {

  public static void main(String[] args) {
    int[] arrInt = new int[]{2,3,1,1,4,2,1};
    System.out.println(new JumpGame().jump(arrInt));
  }

  public int jump(int[] nums) {
    //小于等于1的都不需要跳
    int lengths = nums.length;
    if(lengths <= 1){
      return 0;
    }
    int reach = 0;  //当前能走的最远距离
    int nextreach = nums[0];
    int step = 0;  //需要步数
    for(int i = 0;i<lengths;i++){
      //贪心算法核心：这一步是不是可以比上一步得到更多步数，可以则取最新的路线。
      nextreach = Math.max(i+nums[i],nextreach);
      if(nextreach >= lengths-1) return (step+1);
      if(i == reach){
        step++;
        reach = nextreach;
      }
    }
    return step;
  }

}

```

类似的贪心算法有最优装载问题，活动安排，减绳子等[ref](https://blog.csdn.net/seagal890/article/details/90614064)

重构字符串经常会出现

### 回溯，动规，贪心总结

有这么个问题

给定一个非负整数数组，你最初位于数组的第一个位置。数组中的每个元素代表你在该位置可以跳跃的最大长度。判断你是否能够到达最后一个位置。

输入: [3,2,1,0,4] 输出: false 无论怎样，你总会到达索引为 3 的位置。但该位置的最大跳跃长度是 0 ， 所以你永远不可能到达最后一个位置。

思路：如果我们可以从数组中的某个位置跳到最后的位置，就称这个位置是“好坐标”，否则称为“坏坐标”。问题可以简化为第 0 个位置是不是“好坐标”。

这是一个动态规划问题，通常解决并理解一个动态规划问题需要以下 4 个步骤：

1、利用递归回溯解决问题
2、利用记忆表优化（自顶向下的动态规划）
3、移除递归的部分（自底向上的动态规划）
4、使用技巧减少时间和空间复杂度

方法 1：回溯

这是一个低效的解决方法。我们模拟从第一个位置跳到最后位置的所有方案。从第一个位置开始，模拟所有可以跳到的位置，然后从当前位置重复上述操作，当没有办法继续跳的时候，就回溯。

很标准的回溯解题模板，这个比较好想到，但会超时

```java
public boolean canJumpFromPosition(int position, int[] nums) {
        if (position == nums.length - 1) {
            return true;
        }

        int furthestJump = Math.min(position + nums[position], nums.length - 1);
        for (int nextPosition = position + 1; nextPosition <= furthestJump; nextPosition++) {
            if (canJumpFromPosition(nextPosition, nums)) {
                return true;
            }
        }

        return false;
    }

    public boolean canJump(int[] nums) {
        return canJumpFromPosition(0, nums);
    }

```

方法 2：自顶向下的动态规划

自顶向下的动态规划可以理解成回溯法的一种优化。我们发现当一个坐标已经被确定为好 / 坏之后，结果就不会改变了，这意味着我们可以记录这个结果，每次不用重新计算。

因此，对于数组中的每个位置，我们记录当前坐标是好 / 坏，记录在数组 memo 中，定义元素取值为 GOOD ，BAD，UNKNOWN。这种方法被称为记忆化。

例如，对于输入数组 nums = [2, 4, 2, 1, 0, 2, 0] 的记忆表如下，G 代表 GOOD，B 代表 BAD。我们发现不能从下标 2，3，4 到达最终坐标 6，但可以从 0，1，5 和 6 到达最终坐标 6。

| index | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
| :----: | :----:  | :----: | :----: | :----: | :----: | :----: | :----: |
| nums | 2 | 4 | 2 | 1 | 0 | 2 | 0 |
| memo | G | G | B | B | B | G | G |

步骤

1、初始化 memo 的所有元素为 UNKNOWN，除了最后一个显然是 GOOD （自己一定可以跳到自己）
2、优化递归算法，每步回溯前先检查这个位置是否计算过（当前值为：GOOD / BAD）
	- 如果已知直接返回结果 True / False
	- 否则按照之前的回溯步骤计算
3、计算完毕后，将结果存入memo表中

```java
enum Index {
    GOOD, BAD, UNKNOWN
}

public class Solution {
    Index[] memo;

    public boolean canJumpFromPosition(int position, int[] nums) {
        if (memo[position] != Index.UNKNOWN) {
            return memo[position] == Index.GOOD ? true : false;
        }

        int furthestJump = Math.min(position + nums[position], nums.length - 1);
        for (int nextPosition = position + 1; nextPosition <= furthestJump; nextPosition++) {
            if (canJumpFromPosition(nextPosition, nums)) {
                memo[position] = Index.GOOD;
                return true;
            }
        }

        memo[position] = Index.BAD;
        return false;
    }

    public boolean canJump(int[] nums) {
				//初始化，自顶向下
				memo = new Index[nums.length];
        for (int i = 0; i < memo.length; i++) {
            memo[i] = Index.UNKNOWN;
        }
        memo[memo.length - 1] = Index.GOOD;
				//从第0个位置开始是否可行
        return canJumpFromPosition(0, nums);
    }
}

```

方法 3：自底向上的动态规划

底向上和自顶向下动态规划的区别就是消除了回溯，在实际使用中，自底向下的方法有更好的时间效率因为我们不再需要栈空间，可以节省很多缓存开销。更重要的事，这可以让之后更有优化的空间。回溯通常是通过反转动态规划的步骤来实现的。

这边就要理解一下，一个是要消除递归，消除递归，其实就是干掉回溯，也就是一般会看见两个for循环了，非递归也就一般不要另写函数了

这是由于我们每次只会向右跳动，意味着如果我们从右边开始动态规划，每次查询右边节点的信息，都是已经计算过了的，不再需要额外的递归开销，因为我们每次在 memo 表中都可以找到结果。

```java
enum Index {
    GOOD, BAD, UNKNOWN
}

public class Solution {
    public boolean canJump(int[] nums) {
        Index[] memo = new Index[nums.length];
        for (int i = 0; i < memo.length; i++) {
            memo[i] = Index.UNKNOWN;
        }
        memo[memo.length - 1] = Index.GOOD;

        for (int i = nums.length - 2; i >= 0; i--) {
            int furthestJump = Math.min(i + nums[i], nums.length - 1);
						//只要我的最大距离能有抵达的，那么就是好位置，从最右边动态规划
						for (int j = i + 1; j <= furthestJump; j++) {
                if (memo[j] == Index.GOOD) {
                    memo[i] = Index.GOOD;
                    break;
                }
            }
        }

        return memo[0] == Index.GOOD;
    }
}

```

方法 4：贪心

当我们把代码改成自底向上的模式，我们会有一个重要的发现，从某个位置出发，我们只需要找到第一个标记为 GOOD 的坐标（由跳出循环的条件可得），也就是说找到最左边的那个坐标。如果我们用一个单独的变量来记录最左边的 GOOD 位置，我们就可以避免搜索整个数组，进而可以省略整个 memo 数组。

从右向左迭代，对于每个节点我们检查是否存在一步跳跃可以到达 GOOD 的位置（currPosition + nums[currPosition] >= leftmostGoodIndex）。如果可以到达，当前位置也标记为 GOOD ，同时，这个位置将成为新的最左边的 GOOD 位置，一直重复到数组的开头，如果第一个坐标标记为 GOOD 意味着可以从第一个位置跳到最后的位置。

模拟一下这个操作，对于输入数组 nums = [9, 4, 2, 1, 0, 2, 0]，我们用 G 表示 GOOD，用 B 表示 BAD 和 U 表示 UNKNOWN。我们需要考虑所有从 0 出发的情况并判断坐标 0 是否是好坐标。由于坐标 1 是 GOOD，我们可以从 0 跳到 1 并且 1 最终可以跳到坐标 6，所以尽管 nums[0] 可以直接跳到最后的位置，我们只需要一种方案就可以知道结果。

| index | 0 | 1 | 2 | 3 | 4 | 5 | 6 |
| :----: | :----:  | :----: | :----: | :----: | :----: | :----: | :----: |
| nums | 9 | 4 | 2 | 1 | 0 | 2 | 0 |
| memo | U | G | B | B | B | G | G |

```java
public class Solution {
    public boolean canJump(int[] nums) {
        int lastPos = nums.length - 1;
        for (int i = nums.length - 1; i >= 0; i--) {
            if (i + nums[i] >= lastPos) {
                lastPos = i;
            }
        }
        return lastPos == 0;
    }
}

```

### 分治法


一、基本概念

在计算机科学中，分治法是一种很重要的算法。分治算法，字面上的解释是“分而治之”，分治算法主要是三点：

1.将一个复杂的问题分成两个或更多的相同或相似的子问题，再把子问题分成更小的子问题----“分”

2.将最后子问题可以简单的直接求解----“治”

3.将所有子问题的解合并起来就是原问题打得解----“合”

这三点是分治算法的主要特点，只要是符合这三个特点的问题都可以使用分治算法进行解决(注意用词，是”用”,至于好不好就是另外一回事了)

二、分治法的特征

分治法所能解决的问题一般具有以下几个特征：

1) 该问题的规模缩小到一定的程度就可以容易地解决

2) 该问题可以分解为若干个规模较小的相同问题，即该问题具有最优子结构性质。

3) 利用该问题分解出的子问题的解可以合并为该问题的解；

4) 该问题所分解出的各个子问题是相互独立的，即子问题之间不包含公共的子子问题。

第一条特征是绝大多数问题都可以满足的，因为问题的计算复杂性一般是随着问题规模的增加而增加；

第二条特征是应用分治法的前提它也是大多数问题可以满足的，此特征反映了递归思想的应用；、

第三条特征是关键，能否利用分治法完全取决于问题是否具有第三条特征，如果具备了第一条和第二条特征，而不具备第三条特征，则可以考虑用贪心法或动态规划法。

第四条特征涉及到分治法的效率，如果各子问题是不独立的则分治法要做许多不必要的工作，重复地解公共的子问题，此时虽然可用分治法，但一般用动态规划法较好。

三、为什么用分治法?怎么正确使用分治法?

为什么用分治算法？我们使用一种算法的原因大部分情况下都是为了”快“，只有在少数情况下，在程序已经足够”快“的前提下，我们才会牺牲一部分的”快“，去保全一些开发因素（比如，程序的可维护性等等），那么分治算法为什么快？我们在用这个算法之前必需理解清楚这个问题。

分治算法的思想就是将一个问题规模比较大的问题划分为几个相同逻辑性质（或者直接理解为类似）的问题规模变小的子问题。我们可以从这里入手。

举个超级简单的例子：

假如有一个存在n个元素的int型数组，我们需要求该数组的和。

可能有些人想不想就是一个分治算法，将这个问题分为两个子问题，然后每个子问题再分为两个子问题，当子问题的规模为只有两个数时进行相加。。。

然而，这种办法是使用了分治算法，可是效率比直接遍历一遍相加得到的效率还要低的多.

为什么？因为分治算法本身不适合这种单次遍历就可以搞定的简单问题。你们在阅读一遍分治算法的思想：分治算法的思想就是将一个问题规模比较大的问题划分为几个相同逻辑性质的问题规模变小的子问题，那么这个定义存在一个隐含的前提，当问题规模比较大时，该问题解决起来要成倍的困难！

我们可以举这样一个简单的例子：
我们对一个存在n个元素的数组，使用简单排序进行排序时：

当n=1时，无需比较

当n=2时，我们需要1次比较

当n=3时，我们需要3次比较

当n=4时，我们需要6次比较

当n的数值比较大时，我们需要比较的次数越来越多将会是一个巨大的数字。

而对于前面的求和的例子：
当n=1时，无需相加

当n=2时，我们需要1次相加

当n=3时，我们需要2次相加

当n=4时，我们需要3次相加

仔细观察这组数据，是否发现了什么？

对于求和的例子来说，该问题的计算量与问题规模成正比，在相同的条件下，我们根本无须使用分治算法，因为即使这个问题规模变大，他的解决问题的难易程度没有丝毫改变，它所付出的，只不过是增大了问题规模后所必须付出的计算量，概括起来就是线性增长的问题规模导致了线性增长的计算量。

而对于排序的例子，当问题规模变大时，计算量的增大是成幂次型增长的，概括起来就是线性增长的问题规模导致了幂次型计算量的增长。使得问题规模大的问题解决起来更加困难。

综合起来概括，在问题规模与计算量成正比的算法中，分治算法不是最好的解法，并且有可能是效率极其底下的算法。如果存在某个问题，线性增长的问题规模可能带动计算量的非线性增长，并且符合分治算法的三个特征，那么分治算法是一个很不错的选择。

分治算法很典型的案例：

1、二分查找（递归非递归写法）
2、合并排序
3、快速排序

### 最小表示法和最大表示法

给定字符串，这个字符串是循环的，要求寻找一个位置，以该位置为起点的字符串的字典序在所有的字符串中中最小。

线性算法，不是暴力破解的O(n^2)

开始，i=0，j=1，k=0,其中i，j，k代表以i开头和以j开头的字符串的前k个字符相同

只有三种情况

1.如果str[i+k]==str[j+k] k++。

2.如果str[i+k] > str[j+k] i = i + k + 1，即最小表示不可能以str[i->i+k]开头。

3.如果str[i+k] < str[j+k] j = j + k + 1，即最小表示不可能以str[j->j+k]开头。

那么只要循环n次，就能够判断出字符串的最小表示是以哪个字符开头。

最小表示法

```java
//最小表示法
  public int minRepresent(String s) {
    int n = s.length();
    int i = 0, j = 1, k = 0;
    while (i < n && j < n && k < n) {
      int t = s.charAt((i+k)%n) - s.charAt((j+k)%n);
      if (t == 0) {
        k++;
      } else {
        if (t > 0) {
          i += k+1;
        } else {
          j += k+1;
        }
        if (i == j) {
          j++;
        }
        k = 0;
      }
    }
    return i < j ? i : j;
  }
```

最大表示法

```java
//最大表示法
  public int maxRepresent(String s) {
//    s = s + ".";
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
    return i < j ? i : j;
```

`bcac`这种字串会使最大表示法失效，一个方法是在字串后加一个比'a'小的字串

### LRU缓存算法的实现

LRU（Least Recently Used）是一种常见的页面置换算法，在计算中，所有的文件操作都要放在内存中进行，然而计算机内存大小是固定的，所以我们不可能把所有的文件都加载到内存，因此我们需要制定一种策略对加入到内存中的文件进项选择。常见的页面置换算法有如下几种：

1. LRU 最近最久未使用
2. FIFO 先进先出置换算法 类似队列
3. OPT 最佳置换算法 （理想中存在的）
4. NRU Clock置换算法
5. LFU 最少使用置换算法
6. PBA 页面缓冲算法

LRU的设计原理就是，当数据在最近一段时间经常被访问，那么它在以后也会经常被访问。这就意味着，如果经常访问的数据，我们需要然其能够快速命中，而不常访问的数据，我们在容量超出限制内，要将其淘汰

我们可以选择链表+hash表，hash表的搜索可以达到0(1)时间复杂度，这样就完美的解决我们搜索时间慢的问题了

Hash表，在Java中HashMap是我们的不二选择，链表，Node一个双向链表的实现，Node中存放的是数结构如下：

```java
class Node<K,V>{
	private K key;
	private V value;
	private Node<K,V> prev;
	private Node<K,V> next;
}
```
我们通过HashMap中key存储Node的key,value存储Node来建立Map对Node的映射关系。我们将HashMap看作是一张检索表，我们可以可以快速的检索到我们需要定位的Node，算法思路

1、构建双向链表节点ListNode，应包含key,value,prev,next这几个基本属性

2、对于Cache对象来说，我们需要规定缓存的容量，所以在初始化时，设置容量大小，然后实例化双向链表的head,tail，并让head.next->tail tail.prev->head，这样我们的双向链表构建完成

3、对于get操作,我们首先查阅hashmap，如果存在的话，直接将Node从当前位置移除，然后插入到链表的首部，在链表中实现删除直接让node的前驱节点指向后继节点，很方便.如果不存在，那么直接返回Null

4、对于put操作，比较麻烦。检测hash表，如果存在，插入到链表首部，如果不存在，检测是否超出缓存容量，没有，则创建node并加入链表首部，如果超出容量，就从Hash表key移除链表尾部的node

```java
package com.qiming.algorithm;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class LRUCache<V> {

  /**
   * 容量
   */
  private int capacity = 1024;
  /**
   * Node记录表
   */
  private Map<String, ListNode<String, V>> table = new ConcurrentHashMap<>();
  /**
   * 双向链表头部
   */
  private ListNode<String, V> head;
  /**
   * 双向链表尾部
   */
  private ListNode<String, V> tail;


  public LRUCache(int capacity) {
    this();
    this.capacity = capacity;
  }


  public LRUCache() {
    head = new ListNode<>();
    tail = new ListNode<>();
    head.next = tail;
    head.prev = null;
    tail.prev = head;
    tail.next = null;
  }


  public V get(String key) {

    ListNode<String, V> node = table.get(key);
    //如果Node不在表中，代表缓存中并没有
    if (node == null) {
      return null;
    }
    //如果存在，则需要移动Node节点到表头


    //截断链表，node.prev -> node  -> node.next ====> node.prev -> node.next
    //         node.prev <- node <- node.next  ====>  node.prev <- node.next
    node.prev.next = node.next;
    node.next.prev = node.prev;

    //移动节点到表头
    node.next = head.next;
    head.next.prev = node;
    node.prev = head;
    head.next = node;
    //存在缓存表
    table.put(key, node);
    return node.value;
  }

  /**
   * put操作，检测hash表，如果存在，插入到链表首部，如果不存在，检测是否超出缓存容量，没有，则创建node并加入链表首部，如果超出容量，就从Hash表key移除链表尾部的node
   * @param key
   * @param value
   */
  public void put(String key, V value) {
    ListNode<String, V> node = table.get(key);
    //如果Node不在表中，代表缓存中并没有
    if (node == null) {
      if (table.size() == capacity) {
        //超过容量了 ,首先移除尾部的节点
        table.remove(tail.prev.key);
        //更新tail
        tail.prev.prev.next = tail;
        tail.prev = tail.prev.prev;
      }
      node = new ListNode<>();
      node.key = key;
      node.value = value;
      table.put(key, node);
    }
    //如果存在，则需要移动Node节点到表头
    node.next = head.next;
    head.next.prev = node;
    node.prev = head;
    head.next = node;


  }

  /**
   * 双向链表内部类
   */
  public static class ListNode<K, V> {
    private K key;
    private V value;
    ListNode<K, V> prev;
    ListNode<K, V> next;

    public ListNode(K key, V value) {
      this.key = key;
      this.value = value;
    }


    public ListNode() {

    }
  }


  public static void main(String[] args) {
    LRUCache<ListNode> cache = new LRUCache<>(4);
    ListNode<String, Integer> node1 = new ListNode<>("key1", 1);
    ListNode<String, Integer> node2 = new ListNode<>("key2", 2);
    ListNode<String, Integer> node3 = new ListNode<>("key3", 3);
    ListNode<String, Integer> node4 = new ListNode<>("key4", 4);
    ListNode<String, Integer> node5 = new ListNode<>("key5", 5);
    cache.put("key1", node1);
    cache.put("key2", node2);
    cache.put("key3", node3);
    cache.put("key4", node4);
    cache.get("key2");
    cache.put("key5", node5);
    cache.get("key2");
  }

}

```

### 与，或和异或

| 符号 | 说明 | 运算规则 |
| :----: | :----:  | :----: |
| & | 与 | 两个位都为1时，结果才为1（统计奇数） |
| | | 或 | 两个位都为0时，结果才为0（统计偶数）|
| ^ | 异或 | 两个位相同为0，相异为1（统计不相同数） |


### 面试

1. 面试时要熟练掌握链表，树，栈，队列，哈希表等数据结构，以及他们的操作，链表和二叉树特别多人问，
2. 也会考察查找和排序，重点掌握二分查找，归并排序和快速排序

数据结构一直是技术面试的重点，大多数面试题都是围绕着数组，字符串，链表，树，栈，队列这几种常见的数据结构展开的

一些常见的解题思路：

1. 二维数组的查找，以由上角为标准点，每次删除一列或者删除一行
2. 字符替换，O(n)，从后往前，用1个指针记录字符串末尾，用一个指针记录替换后的字符串末尾
3. 逆向的打印链表中的值，其实是后进先出，每经过一个节点，将值放入栈中，既然想到栈，也可以用递归来解决，但如果链表非常长时，函数调用的层级很深
4. 树有三种遍历方式，前序，中序，后序，而这三种遍历又都有递归和循环两种实现方式，共6种实现方法，**根据前序遍历和中序遍历进行树的重构**
5. 用两个栈实现队列，在第二栈为空时，删除的时候把数据都弹过去，只要第二栈有数据，删除就在第二个栈中，插入仍在第一个栈中
6. 反过来，用两个队列实现一个栈，删除头部的数据到第二个队列，然后就可以删除原队列中的头结点，以后每次都是这样操作
7. 二分查找(折半查找)
8. 归并排序
9. 快速排序
10. 排序会去比较插入排序，冒泡排序，归并排序和快速排序等不同算法的优劣
11. 位运算
12. 在O(1)的时间删除链表结点，链表类型的题目，要看好指针
13. 合并两个升序的链表，比较头部，头节点开始
14. 出现的次数超过数组长度的一半的数
15. 找出最小的K个数，可以用partition函数
16. 丑数，只包含因子2、3、5的数称作丑数，求按从小到大的顺序的第1500个丑数，例如6、8都是丑数，但14不是，因为它包含因子7，习惯上我们把1当做是第一个丑数
17. 第一个只出现一次的字符，不要写(On*n)的
18. 两个链表的第一个公共结点
19. 数字在排序数组中出现的次数
20. 数组中只出现一次的数
21. 不用加减乘除做加法，写一个函数，求两个整数的和，要求在函数体内不得使用加减乘除四则运算
22. 二叉搜索树中第K小的元素，这个题目实际上就是中序遍历二叉树，然后找第k个元素

### 补充

1. 为什么很多程序竞赛题目都要求答案对 1e9+7 取模？
	- 其实不止1e9+7，还有1e9+9和998244353。这三个数都是一个质数，同时小于 [公式] 。所以有什么好处呢？
	- 所有模过之后的数在加法操作在int范围内不会溢出，即a,b < 2^30 , a + b < 2^31
	- 在乘法操作后在long long范围内不会溢出，即ab < 2^60
