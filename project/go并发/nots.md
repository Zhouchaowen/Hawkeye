- 基本并发原语:在这部分，我会介绍Mutex. RWMutex、 Waitgroup、 Cond、 Pool、Context等标准库中的并发原语，这些都是传统的并发原语，在其它语言中也很常见，是我们在并发编程中常用的类型。

- 原子操作:在这部分,我会介绍Go标准库中提供的原子操作。原子操作是其它并发原语的基础，学会了你就可以自己创造新的并发原语。

- Channel: Channel 类型是Go语言独特的类型，因为比较新，所以难以掌握。但是别怕，我会带你全方位地学习Channel类型,你不仅能掌握它的基本用法,而且还能掌握它的处理场景和应用模式，避免踩坑。
- 扩展并发原语:目前来看，Go开发组不准备在标准库中扩充并发原语了,但是还有一 些并发原语应用广泛,比如信号量、SingleFlight. 循环栅栏、ErrGroup等。 掌握了它们，就可以在处理一些并发问题时，取得事半功倍的效果。
- 分布式并发原语:分布式并发原语是应对大规模的应用程序中并发问题的并发类型。我主要会介绍使用etcd实现的一-些纷布式并发原语,比如Leader选举、分布式互斥锁、分布式读写锁、分布式队列等,在处理分布式场景的并发问题时,特别有用。



**并发安全产生的原因：多个goroutine并发更新同一个资源,像计数器;**

## 互斥锁

- 介绍：

临界区：在并发编程中，如果程序中的一部分会被并发访问或修改，那么，为了避免并发访问导致的意想不到的结果，这部分程序需要被保护起来，这部分被保护起来的程序，就叫做临界区。可以说，临界区就是-个被共享的资源,或者说是一个 整体的一组共享资源，比如对数据库的访问、对某一个共享 数据结构的操作、对一个I/O设备的使用、对一个连接池中的连接的调用，等。

![image-20220418122011015](C:\Users\zcw\AppData\Roaming\Typora\typora-user-images\image-20220418122011015.png)

- 问题案例：

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var count = 0
	//使用WaitGroup等待10个goroutine完成
	var wg sync.WaitGroup
	for i := 0;i<10;i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0;j<10000;j++ {
				count++
			}
		}()
	}
	wg.Wait()
	fmt.Println(count)
}

/**
ch1> go run .\main.go
47669
/
```

- 产生原因：

实,这是因为，count++ 不是一个原子操作， 它至少包含几个步骤，比如读取变量count的当前值，对这个值加1,把结果再保存到count中。因为不是原子操作，就可能有并发的问题。

Go race detector基于Google的C/C+ + 技术实现的，编译器通过探测所有的内存访问，加入代码能监视对这些内存地址的访问(读还是写)。在代码运行的时候,race detector 就能监控到对共享变量的非同步访问，出现race的时候，就会打印出警告信息。

虽然这个具使用起来很方便，但是，因为它的实现方式,只能通过真正对实际地址进行读写访问的时候才能探测，所以它并不能在编译的时候发现data race的问题。而且，在运行的时候，只有在触发了data race之后，才能检测到，如果碰巧没有触发(此如一个data race问题只能在2月14号零点或者11月11号零点才出现)，是检测不出来的。而且，把开启了race 的程序部署在线上，还是比较影响性能的。运行go tool compile -race -S counter.go,可以查看计数器例子的代码，重点关注一下 count+ +前后的编译后的代码:

- 解决方法：

```go
// ch1
package main

import (
	"fmt"
	"sync"
)


func main() {
	var count = 0
	//使用WaitGroup等待10个goroutine完成
	var wg sync.WaitGroup
	var mu sync.Mutex
	for i := 0;i<10;i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0;j<10000;j++ {
				mu.Lock()
				count++
				mu.Unlock()
			}
		}()
	}
	wg.Wait()
	fmt.Println(count)
}
```

```go
// ch2
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var count int64 = 0
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				atomic.AddInt64(&count, 1)
			}
		}()
	}
	wg.Wait()
	fmt.Println("count的值:",count)
}
```

```go
// ch3
package main

import (
	"fmt"
	"sync"
)

func main() {
	ch := make(chan struct{}) // 发送和接收count++"信号"的通道

	wg := sync.WaitGroup{}
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < 100000; j++ {
				ch <- struct{}{}
			}
		}()
	}

	go func() {
		wg.Wait() // 等待上面所有的 goroutine 运行完成
		close(ch) // 关闭ch通道
	}()

	count := 0
	for range ch { // 如果ch通道读取完了(ch是关闭状态), 则for循环结束
		count++
	}
	fmt.Println("count的值是:", count)
}
```

- 思考：

如果 Mutex 已经被一个 goroutine 获取了锁，其它等待中的 goroutine 们只能一直等待。那么，等这个锁释放后，等待中的 goroutine 中哪一个会优先获取 Mutex 呢？

等待的goroutine们是以FIFO排队的

1）当Mutex处于正常模式时，若此时没有新goroutine与队头goroutine竞争，则队头goroutine获得。若有新goroutine竞争大概率新goroutine获得。 

2）当队头goroutine竞争锁失败1ms后，它会将Mutex调整为饥饿模式。进入饥饿模式后，锁的所有权会直接从解锁goroutine移交给队头goroutine，此时新来的goroutine直接放入队尾。 

3）当一个goroutine获取锁后，如果发现自己满足下列条件中的任何一个

​		1它是队列中最后一个

​		2它等待锁的时间少于1ms，则将锁切换回正常模式

https://golang.org/src/sync/mutex.go

详解

演进历史

”初版”的Mutex使用一个flag来表示锁是否被持有，实现比较简单;后来照顾到新来的goroutine,所以会让新的goroutine也尽可能地先获取到锁，这是第二个阶段，我把它叫作”给新人机会”;那么，接下来就是第三阶段“多给些机会”， 照顾新来的和被唤醒的goroutine;但是这样会带来饥饿问题，所以目前又加入了饥饿的解决方案,也就是第四阶段“解决饥饿”。

初版的Mutex实现有一个问题: 请求锁的goroutine会排队等待获取互斥锁。虽然这貌似很公平,但是从性能上来看，却不是最优的。因为如果我们能够把锁交给正在占用CPU时间片的goroutine的话，那就不需要做上下文的切换，在高并发的情况下，可能会有好的性能。