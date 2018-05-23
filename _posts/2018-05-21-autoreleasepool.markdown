---
layout:     post
title:      "autoreleasepool"
subtitle:   ""
date:       2018-05-21
author:     "李书勋"
header-img: "img/post-sample-image.jpg"
tags:
    - iOS
    - 内存管理
---

## Catagory
   1. [Autorelease Pool Blocks](#autorelease-pool-blocks)
   2. [AutoreleasePoolPage](#autoreleasepoolpage)
   3. [Push操作](#push操作)
        * [add](#add)
        * [autoreleaseFullPage](#autoreleasefullpage)
        * [autoreleaseNoPage](#autoreleasenopage)
   4. [Autorelease](#autorelease)
   5. [Pop](#pop)
        * [autorelease顺序图](#autorelease顺序图)
       
---

##  Autorelease Pool Blocks

> 我们使用`clang -rewrite-objc`命令将OC code 重写为C++code，会出现这段部分结果。

```objc
/* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 

}
```
 ![autoreleasepool结构图](http://chuantu.biz/t6/316/1526973583x-1566638189.png)
 
 不得不说，苹果对 `@autoreleasepool {}` 的实现真的是非常巧妙，通过声明一个 `__AtAutoreleasePool`类型的局部变量 `__autoreleasepool`来实现 `@autoreleasepool {} `。当声明 `__autoreleasepool` 变量时，构造函数 `__AtAutoreleasePool()` 被调用，即执行 `atautoreleasepoolobj = objc_autoreleasePoolPush();`；当出了当前作用域时，析构函数 `~__AtAutoreleasePool()`被调用，即执行 `objc_autoreleasePoolPop(atautoreleasepoolobj);`。所以`@autoreleasepool {}`的实现代码可以进一步简化如下：
 ```objc
 @autoreleasepool {
 void *atautoreleasepoolobj = objc_autoreleasePoolPush();
 // 用户代码，所有接收到 autorelease 消息的对象会被添加到这个 autoreleasepool 中
 
 objc_autoreleasePoolPop(atautoreleasepoolobj);
 }
 ```
 因此，单个 `autoreleasepool` 的运行过程可以简单地理解为`objc_autoreleasePoolPush()`、`[objc autorelease]` 和 `objc_autoreleasePoolPop(void *)`三个过程。
 
 ---
 
## AutoreleasePoolPage
 
 `-[NSAutoreleasePool release]`最终是通过调用`AutoreleasePoolPage::pop(void *)`函数来负责对 `autoreleasepool` 中的 `autoreleased` 对象执行 `release` 操作的。
 
 根据源码我们发现，`autoreleasepool`是没有单独的内存结构的，它是通过以 `AutoreleasePoolPage`为结点的双向链表来实现的。
 
 ![AutoreleasePoolPage源码](http://chuantu.biz/t6/316/1526976021x-1566638189.png)
 
1. magic 用来校验 AutoreleasePoolPage 的结构是否完整；
2. next 指向最新添加的 autoreleased 对象的下一个位置，初始化时指向 begin() ；
3. thread 指向当前线程；
4. parent 指向父结点，第一个结点的 parent 值为 nil ；
5. child 指向子结点，最后一个结点的 child 值为 nil ；
6. depth 代表深度，从 0 开始，往后递增 1；
7. hiwat 代表 high water mark 。
8. AutoreleasePoolPage每个对象会开辟4096字节内存（也就是虚拟内存一页的大小），除了上面的实例变量所占空间，剩下的空间全部用来储存autorelease对象的地址。
9. 一个AutoreleasePoolPage的空间被占满时，会新建一个AutoreleasePoolPage对象，连接链表，后来的autorelease对象在新的page加入。
 
 ![page结构图](http://chuantu.biz/t6/317/1527058386x-1566638189.png)
 
 **向一个对象发送- autorelease消息，就是将这个对象加入到当前AutoreleasePoolPage的栈顶next指针指向的位置**
 
 ---
 
## Push操作
  `objc_autoreleasePoolPush()` 函数本质上就是调用的 `AutoreleasePoolPage` 的 `push` 函数。    
 ![PoolPush()](http://chuantu.biz/t6/316/1526975346x-1566638189.png)
 ![Push函数](http://chuantu.biz/t6/316/1526975321x-1566638189.png)
其实就是创建一个新的 autoreleasepool ，对应 AutoreleasePoolPage 的具体实现就是往 AutoreleasePoolPage 中的 `next` 位置插入一个 `POOL_BOUNDARY` 哨兵对象，并且返回插入的 `POOL_BOUNDARY` 的内存地址。
 ![autoreleaseFast()函数](http://chuantu.biz/t6/316/1526976930x-1566638189.png)
 `autoreleaseFast`函数在执行一个具体的插入操作时，分别对三种情况进行了不同的处理：
 
 1. 当前 page 存在且没有满时，直接将对象添加到当前 page 中，即 next 指向的位置；
 2. 当前 page 存在且已满时，创建一个新的 page ，并将对象添加到新创建的 page 中；
 3. 当前 page 不存在时，即还没有 page 时，创建第一个 page ，并将对象添加到新创建的 page 中。     
 结果: 每调用一次 push 操作就会创建一个新的 `autoreleasepool` ，即往`AutoreleasePoolPage` 中插入一个 `POOL_BOUNDARY` ，并且返回插入的`POOL_BOUNDARY`的内存地址。
 
 - ### add
    ![add函数](http://chuantu.biz/t6/317/1526978285x-1566638189.png)
    首先 解除保护， 然后将 objc 入栈到栈顶并重新定义栈顶， 添加保护。
 - ### autoreleaseFullPage
    ![autoreleaseFullPage函数](http://chuantu.biz/t6/317/1526978267x-1566638189.png)
 - ### autoreleaseNoPage
    ![autoreleaseNoPage函数1](http://chuantu.biz/t6/317/1526978316x-1566638189.png)
    ![autoreleaseNoPage函数2](http://chuantu.biz/t6/317/1526978329x-1566638189.png)
    
 ---

## Autorelease

- ### autorelease
我们看到`autorelease`函数的具体实现， 然后和push操作很像，但是`autorelease`函数入栈的是具体的对象，而不是一个哨兵对象。
![autorelease函数](http://chuantu.biz/t6/317/1526980565x-1566638189.png)
![rootAutorelease函数](http://chuantu.biz/t6/317/1526980552x-1566638189.png)
![rootAutorelease2函数](http://chuantu.biz/t6/317/1526980538x-1566638189.png)
![autorelease实际函数](http://chuantu.biz/t6/317/1526980525x-1566638189.png)

---

## Pop

`objc_autoreleasePoolPop(void *)`函数本质上也是调用的 `AutoreleasePoolPage` 的 `pop` 函数。

```objc
void
objc_autoreleasePoolPop(void *ctxt)
{
AutoreleasePoolPage::pop(ctxt);
}
```

![pop函数1](http://chuantu.biz/t6/317/1527055533x-1566638189.png)
![pop函数2](http://chuantu.biz/t6/317/1527055547x-1566638189.png)

- ### autorelease顺序图

每当进行一次`objc_autoreleasePoolPush`调用时,`runtime`向当前的`AutoreleasePoolPage`中add进一个哨兵对象，值为0（也就是个nil），那么这一个page就变成了下面的样子：

![push结构图](http://chuantu.biz/t6/317/1527058882x-1566638189.png)

`objc_autoreleasePoolPush`的返回值正是这个哨兵对象的地址，被`objc_autoreleasePoolPop`(哨兵对象)作为入参，于是：

1. 根据传入的哨兵对象地址找到哨兵对象所处的page
2. 在当前page中，将晚于哨兵对象插入的所有`autorelease`对象都发送一次`-release`消息，并向回移动next指针到正确位置.从最新加入的对象一直向前清理，可以向前跨越若干个page，直到哨兵所在的page.

刚才的`objc_autoreleasePoolPop`执行后，最终变成了下面的样子：
![pop后结构图](http://chuantu.biz/t6/317/1527058812x-1566638189.png)

---

[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)   [Objective-C Autorelease Pool 的实现原理](http://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/#jtss-tsina)
 
---


