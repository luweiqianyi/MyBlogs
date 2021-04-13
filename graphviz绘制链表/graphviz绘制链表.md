# graphviz绘制链表

## 写稿目的

本来想偷懒，对于链表的绘制，想在A4纸张上进行绘制，然后拍个照片作为电子资料，然后存储起来，但是考虑到本人字体“捉鸡”，也不想在相册中保存或者将照片上传到某一硬件介质进行存储，也不可能将A4值作为材料保存（因为考虑到纸质资料不能够随身携带，不可能将所有带有笔记的纸质资料都带在身上)，所以，通过本篇的学习，来绘制基本的链表，之后便可以将由软件绘制出来的图片集成到支持markdown语法的文档中，这样就完成了一篇比较美观的电子化资料了。

## 开发环境准备

* Visual Studio Code

* 安装[Graphviz](https://graphviz.org/download/),因为我的是Windows系统，所以选择Windows版本的安装即可。安装之后需要在系统的环境变量的path路径下，添加dot.exe可执行文件的所在路径。

* 然后在Visual Studio Code上安装相应的支持库和语言

  ![](.\imgs\dev-environment.png)

* 节点可视化准备

  * Ctrl+Shift+P, 在打开的窗口中找到，Graphviz:Open Preview to the Side，点击打开窗口即可，如下

    ![](.\imgs\dev-2window-prepare.png)

## 代码实例分析

### 背景介绍

本来我是在学习Android的消息机制时碰到了`MessageQueue`，一直想知道其内部关于“消息是如何进入队列中”的具体实现，所以去查看了它内部的原理，主要是类`MessageQueue`中的`enqueueMessage`函数,经过分析发现其内部是维护了一个链式链表，以新的想加入这个链式表中的消息的when值与链式表中的其他消息的when值进行比较，然后按照规则加入链式表中，具体代码如下：

```java
1  boolean enqueueMessage(Message msg, long when) {
2          if (msg.target == null) {
3              throw new IllegalArgumentException("Message must have a target.");
4          }
5          if (msg.isInUse()) {
6              throw new IllegalStateException(msg + " This message is already in use.");
7          }
8
9          synchronized (this) {
10              if (mQuitting) {
11                  IllegalStateException e = new IllegalStateException(
12                          msg.target + " sending message to a Handler on a dead thread");
13                  Log.w(TAG, e.getMessage(), e);
14                  msg.recycle();
15                  return false;
16              }
17
18              msg.markInUse();
19              msg.when = when;
20              Message p = mMessages;
21              boolean needWake;
22              if (p == null || when == 0 || when < p.when) {
23                  // New head, wake up the event queue if blocked.
24                  msg.next = p;
25                  mMessages = msg;
26                  needWake = mBlocked;
27              } else {
28                  // Inserted within the middle of the queue.  Usually we don't have to wake
29                  // up the event queue unless there is a barrier at the head of the queue
30                  // and the message is the earliest asynchronous message in the queue.
31                  needWake = mBlocked && p.target == null && msg.isAsynchronous();
32                  Message prev;
33                  for (;;) {
34                      prev = p;
35                      p = p.next;
36                      if (p == null || when < p.when) {
37                          break;
38                      }
39                      if (needWake && p.isAsynchronous()) {
40                          needWake = false;
41                      }
42                  }
43                  msg.next = p; // invariant: p == prev.next
44                  prev.next = msg;
45              }
46  
47              // We can assume mPtr != 0 because mQuitting is false.
48              if (needWake) {
49                  nativeWake(mPtr);
50              }
51          }
52          return true;
53}
```

其中，我关注的是消息是如何进入这个队列中的，所以，看代码的时候可以将一些状态标记和异常检查忽略掉，比如`if (msg.target == null)`、`if (msg.isInUse()) `、`mQuitting、msg.markInUse()、needWake`等，只需要关注line19到line45即可。

* 注意：`mMessages`是一个类型为Message的成员变量。

* 对于Message类，我们目前只需要关注以下内容即可

  ```java
  public final class Message implements Parcelable{
  	long when;
      Message next;
      // 以下省略其他的成员变量和方法
  }
  ```

  用`Graphviz`绘制出来如下：

  ```groovy
  digraph {
      label = "\n msg";
      rankdir = LR; // 布局从左到右
      node [shape = record]; 
      //list [label = "{<s0> when | <s1> next}"]横向
      list [label = "<s0> when | <s1> next"] //链表元素的排列顺序:纵向
      listnode0 [label = "<s0>null"]; // 声明一个空节点
  
      list:s1 -> listnode0 // 指定节点的指向
  
  }
  
  ```

  ![](.\imgs\Message.png)

接下来，我们先看`enqueueMessage`的第一部分，即line22到line27,分三种情况考虑

* p == null
* when == 0
* when < p.when

#### 先考虑三种特殊情况

**备注：以下粉红色的节点表示为我们新加入的节点msg**

##### p==null的情况

**以下是p==null时执行代码line20到line26的情况。**

* 起初`mMessages`的节点情况如下：

  ![](.\imgs\p == null 1.PNG)

* 执行函数`enqueueMessage`中line24之后，情况如下：

  ![](.\imgs\设置msg的next为p.PNG)

* 执行代码line25之后，情况如下：

  ![](.\imgs\设置mMessage指向新加入的节点msg.PNG)

##### when == 0 || when < p.when

出现此情况的前提是p == null 这个条件不成立，所以`mMessages`中至少有一个节点，同时这两种情况都是将待插入节点msg添加到链表头部的情况，所以以下分一个节点和多个节点2中情况进行新节点插入的分析:

* 初始情况如下，分单个节点和多个节点：

  ![](.\imgs\初始情况-单个节点.PNG)

  ![](.\imgs\初始情况-多个节点.PNG)

* 执行函数`enqueueMessage`中line24之后，情况如下：

  ![](.\imgs\单节点情况下设置新节点的next为p.PNG)

  ![](.\imgs\多节点情况下设置新节点的next为p.PNG)

* 执行代码line25之后，情况如下：

  ![](.\imgs\单节点情况下设置mMessage指向新加入的节点.PNG)

  ![](.\imgs\多节点情况下设置mMessage指向新加入的节点.PNG)

#### 再考虑`enqueueMessage`line32到line44

该情况指的是，`mMessages`中至少存在一个节点，且待新加入的节点msg不加在链表头部的情况。

根据对“when == 0 || when < p.when”情况的分析，我们知道，单节点和多节点的插入逻辑其实是一样的，所以下面使用多节点的方式对代码进行分析。

for循环中line36到line38是for循环的终止条件，表示待加入节点msg一直和这个链表中的其他节点的when变量进行比较，如果msg节点的when值比链表中的某个节点的when值小活着到链表的结束位置还没在链表中找到一个节点(它的when值比msg节点的when值大或相等)，则结束循环。

以下以for循环的第一次执行进行节点图的绘制，其他执行类推即可：

* prev = p

  ![](.\imgs\多节点 prev=p.PNG)

* p = p->next

  ![](.\imgs\多节点p=p的next.PNG)

此时，我们假设待加入节点msg的when的值满足for循环中line36到line38的循环终止条件，则接下来跳出循环，执行line43和line44两行代码，其节点图如下所示

* msg.next = p
  
  ![](.\imgs\多节点msg的next指向p.PNG)
  
* prev.next = msg

  ![](.\imgs\多节点情况下设置prev的next指向新插入的节点msg.PNG)

## 参考

* [Graphviz Document](https://graphviz.org/doc/info/shapes.html#record)

