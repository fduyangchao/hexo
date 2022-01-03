---
title: 两个goroutine交叉打印奇偶数字
date: 2021-10-13 18:38:19
categories: 
  - 专业
  - 开发
  - golang
tags:
  - golang
  - 算法
index_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/golang-5.png
banner_img: https://cdn.jsdelivr.net/gh/simonyangchao/resources@image/golang-5.png
---

之前面试被问到过一个比较简单的进程同步场景：如何使用两个进程（golang就是两个go routine) 交叉打印1-100这100个数字。在作为面试官的时候也问过候选人这个问题，但是没有深入总结过这个问题有多少解决方案。本文主要记录一下分析过程。

<!-- more -->

## 解答思路

第一反应是用两个无缓冲channel实现两个go routine之间的信息交互，互相阻塞实现同步。

```
package main

import (
	"fmt"
	"sync"
)

func main() {

	wg := sync.WaitGroup{}

	wg.Add(2)

	channel1 := make(chan struct{})
	channel2 := make(chan struct{})

	go func() {
		defer wg.Done()
		for i := 0; i <= 100; i++ {
			if i%2 != 0 {
				continue
			}
			<-channel2
			fmt.Printf("goroutine a: %d\n", i)
			channel1 <- struct{}{}
		}
	}()

	go func() {
		defer wg.Done()
		for i := 0; i <= 100; i++ {
			if i%2 == 0 {
				continue
			}
			<-channel1
			fmt.Printf("goroutine b: %d\n", i)
			channel2 <- struct{}{}
		}
	}()

	wg.Wait()
}
```

但是这种解法有明显的问题：两个go routine互相死锁。原因显而易见，两个channel互相阻塞导致。

```
chao@chao:~/code/src/demo$ go run main.go
fatal error: all goroutines are asleep - deadlock!
```

那么，只用一个channel能否解决这个问题？

```
func main() {

	wg := sync.WaitGroup{}

	wg.Add(2)

	channel := make(chan struct{})

	go func() {
		defer wg.Done()
		for i := 0; i <= 100; i++ {
			if i%2 != 0 {
				continue
			}
			fmt.Printf("goroutine a: %d\n", i)
			channel <- struct{}{}
		}
	}()

	go func() {
		defer wg.Done()
		for i := 0; i <= 100; i++ {
			if i%2 == 0 {
				continue
			}
			<-channel
			fmt.Printf("goroutine b: %d\n", i)
		}
	}()

	wg.Wait()
}
```

```
chao@chao:~/code/src/demo$ go run main.go
goroutine a: 0
goroutine a: 2
goroutine b: 1
goroutine b: 3
goroutine a: 4
goroutine a: 6
goroutine b: 5
goroutine b: 7
goroutine a: 8
goroutine a: 10
goroutine b: 9
goroutine b: 11
...
goroutine a: 100
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [semacquire]:
```

乍一看解决方案清晰明了，但是运行结果不尽如人意：

1. 每个go routine会两个两个交叉打印
2. 最终还是有死锁问题出现

接下来从go routine调度的过程来分析这种方案的错误。首先需要了解go routine的调度逻辑：

在谈goroutine之前，我们先谈谈并发和并行。

一般的程序，如果没有特别的要求的话，是顺序执行的，这样的程序也容易编写维护。但是随着科技的发展、业务的演进，我们不得不变写可以并行的程序，因为这样有很多好处。

比如你在看文章的时候，还可以听着音乐，这就是系统的并行，同时可以做多件事情，充分的利用计算机的多核，提升的软件运行的性能。

在操作系统中，有两个重要的概念：一个是进程、一个是线程。当我们运行一个程序的时候，比如你的IDE或者QQ等，操作系统会为这个程序创建一个进程，这个进程包含了运行这个程序所需的各种资源，可以说它是一个容器，是属于这个程序的工作空间，比如它里面有内存空间、文件句柄、设备和线程等等。

那么线程是什么呢？线程是一个执行的空间，比如要下载一个文件，访问一次网络等等。线程会被操作系统调用，来在不同的处理器上运行编写的代码任务，这个处理器不一定是该程序进程所在的处理。操作系统过的调度是操作系统负责的，不同的操作系统可能会不一样，但是对于我们程序编写者来说，不用关心，因为对我们都是透明的。

一个进程在启动的时候，会创建一个主线程，这个主线程结束的时候，程序进程也就终止了，所以一个进程至少有一个线程，这也是我们在`main`函数里，使用goroutine的时候，要让主线程等待的原因，因为主线程结束了，程序就终止了，那么就有可能会看不到goroutine的输出。

在一个并发指的是将

go语言中并发指的是让某个函数独立于其他函数运行的能力，一个goroutine就是一个独立的工作单元，Go的runtime（运行时）会在**逻辑处理器**上调度这些goroutine来运行，一个**逻辑处理器**绑定一个操作系统线程，所以说goroutine不是线程，它是一个协程，也是这个原因，它是由Go语言运行时本身的算法实现的。

这里我们总结下几个概念：

| 概念         | 说明                                          |
| :----------- | :-------------------------------------------- |
| 进程         | 一个程序对应一个独立程序空间                  |
| 线程         | 一个执行空间，一个进程可以有多个线程          |
| 逻辑处理器   | 执行创建的goroutine，绑定一个线程             |
| 调度器       | Go运行时中的，分配goroutine给不同的逻辑处理器 |
| 全局运行队列 | 所有刚创建的goroutine都会放到这里             |
| 本地运行队列 | 逻辑处理器的goroutine队列                     |

当我们创建一个goroutine的后，会先存放在`全局运行队列`中，等待Go运行时的`调度器`进行调度，把他们分配给其中的一个`逻辑处理器`，并放到这个逻辑处理器对应的`本地运行队列`中，最终等着被`逻辑处理器`执行即可。

这一套管理、调度、执行goroutine的方式称之为Go的并发。并发可以同时做很多事情，比如有个goroutine执行了一半，就被暂停执行其他goroutine去了，这是Go控制管理的。所以并发的概念和并行不一样，并行指的是在不同的物理处理器上同时执行不同的代码片段，**并行可以同时做很多事情，而并发是同时管理很多事情**，因为操作系统和硬件的总资源比较少，所以并发的效果要比并行好的多，使用较少的资源做更多的事情，也是Go语言提倡的。

Go的并发原理我们刚刚讲了，那么Go的并行是怎样的呢？其实答案非常简单，多创建一个`逻辑处理器`就好了，这样调度器就可以同时分配`全局运行队列`中的goroutine到不同的`逻辑处理器`上并行执行。



基于上述对goroutine调度的了解，分析为上述解法会报错：第一个执行周期，goroutine A打印“goroutine a: 0”之后，A和B都处于阻塞状态；这时A向channel中写值，A和B都解除阻塞状态；此时，期望的结果是调度器执行B，打印“goroutine b: 1”，然而调度器的行为不可知，在下一个执行周期可能会执行A，在continue之后打印“goroutine a: 2”，如下图：

![](/images/goroutine-错误方法.png)

正确的channel阻塞方案如下：

```
func main() {

	wg := sync.WaitGroup{}

	wg.Add(2)

	channel := make(chan struct{})

	go func() {
		defer wg.Done()
		for i := 0; i <= 100; i++ {
			channel <- struct{}{}
			if i%2 != 0 {
				continue
			}
			fmt.Printf("goroutine a: %d\n", i)
		}
	}()

	go func() {
		defer wg.Done()
		for i := 0; i <= 100; i++ {
			<-channel
			if i%2 == 0 {
				continue
			}
			fmt.Printf("goroutine b: %d\n", i)
		}
	}()

	wg.Wait()
}

```

这种方案两个go routine刚进入循环的时候都会被阻塞，然后每个loop周期轮流bypass和打印，如下图：

![](/images/goroutine-正确方法.png)



## 参考资料

https://www.flysnow.org/2017/04/11/go-in-action-go-goroutine.html

Golang 调度器 GMP 原理与调度全分析：https://learnku.com/articles/41728
