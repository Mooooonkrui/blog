---
title: "[OSOW-S1EP1] Actor模型的C语言实现"
date: 2025-09-14 19:43:02
tags: [C/C++, C, 并发, Actor]
---

## Cocurrence on the Begining

> 计算机科学之开始，许多范式尚未确立，使用的时候尚须用指针指指点点。

在计算机科学的上古年代，或许不只是上古，mutex、semaphore、conditions、monitor、atomic一直是并发编程中的常客。并发意味着数据的流动和共享，由此带来了数据安全与竞争的风险，轻则出错，重则死锁。程序员往往需要精细地操作每一个数据的所有权和生命周期，规划复杂的竞争条件，甚至又是需要经过形式化验证，才能构造出一个可堪一用的并发系统。每一个计算机学生应该都知道，仅仅是一个无比简单的Reader-Writer问题，也需要使用semaphore甚至conditions才能使之正常运行。

随着计算机硬件的发展，我们有能力使用代价更昂贵的编程方式来进行项目开发，一些曾经不利于性能考量，但是能够大幅提升开发效率的范式也渐渐引起人们的重视。Carl Hewitt在1973年定义的Actor模型就是其中一种。我们可以认为，Actor=数据+行为+信息。Actor完全采用消息模型，将每一个信息处理单元抽象为一个Actor对象，对象存储有一定数据，所有信息通过消息管道在Actor间传递，每个Actor同时只处理一个数据。通过抛弃显式的共享内存调用，Actor避免了所有可能的数据竞争问题。这是以一定性能损失为代价的，但是其带来的开发效率和安全保证是值得的。

Actor模型可以在绝大多数语言中使用。通过消息队列，OOP甚至是手写共享内存管理，大多数语言都提供了Actor模型的库，比如Rust的Actix。不过，Actor模型的最经典实现来自于为其提供了语言层面支持的Erlang。Erlang之于Actor，好比Go之于coroutine。

## Anatomy of Actor

Actor模型由Actor（废话）和Actor之间的消息路径组成，总的来说是一张图。对于每个Actor，我们需要考虑其持有的数据，以及获得消息时的回调函数。简单来说可以这样写：

```c
#define UNIV void*

struct ACTOR {
	UNIV data;
	(UNIV, UNIV)UNIV handler;
};
```

当一个`ACTOR`获得一个数据（由于C无OOP，只能用`UNIV`表示泛型）时，它调用`handler`。`handler`接受传入的数据和自身数据的引用，进行一系列操作，然后给出一个返回值给一个守护进程（涉及资源的创建和回收）。

我们要处理的主要是以下几个问题：

+ `ACTOR`需要能够被创建和销毁，我们需要一个守护进程来管理这些过程。
+ 消息必须是只读的，所有信息通过拷贝生成，使用后销毁，不涉及任何拷贝。
+ `ACTOR`内的`data`只能由自身的`handler`来读写。
+ etc.

## C in Practice

首先我们定义一些宏：

```c
#define UNIV void*
#define HDLR (UNIV, UNIV)UNIV
#define TO_UNIV(x) ((UNIV)(x))
#define TO_HDLR(x) ((HDLR)(x))
```

然后，我们构建出`ACTOR`的基本模型：

```
struct ACTOR {
	UNIV data = nullptr;
	HDLR hdlr = nullptr;
};

ACTOR* new_actor(UNIV data, HDLR hdlr) {
	ACTOR* res = (ACTOR*)malloc(sizeof(ACTOR));
	res.data = data;
	res.hdlr = hdlr;
	return res;
}

#define NEW_ACTOR(data, hdlr) new_actor(TO_UNIV(data), TO_HDLR(hdlr))
```
