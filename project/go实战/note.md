# 提升之路

- 积累代码50万行
- 看英文文档
- 持续锻炼口才，演讲
- 持续输出，公众号，个人品牌
- github看多star项目
  - github Trending，reddit，medium，hacker news，morning，paper，acm.org,oreily



理解可执行文件

```bash
go build -x main.go
```

# Go语言的runtime

## Scheduler
调度器管理所有的G，M, P，在后台执行调度循

![image-20211226140851648](C:\Users\zcw\AppData\Roaming\Typora\typora-user-images\image-20211226140851648.png)

### 调度循环

go后事如何运行的

```go
go func(){
    println("hello world")
}
```

![image-20211226141449786](C:\Users\zcw\AppData\Roaming\Typora\typora-user-images\image-20211226141449786.png)

GMP

![image-20211226141957385](C:\Users\zcw\AppData\Roaming\Typora\typora-user-images\image-20211226141957385.png)



1.生成一个G

![image-20211226142213436](C:\Users\zcw\AppData\Roaming\Typora\typora-user-images\image-20211226142213436.png)

本地队列满后，会把本地队列的一半G拿出来，在加上从runnext踢出来的G。变成链表后拼接到全家队列

2.消费一个G

![image-20211226142832143](C:\Users\zcw\AppData\Roaming\Typora\typora-user-images\image-20211226142832143.png)

调度循环

- 调度到61的倍数时，从全局拿

- 不是61的倍数次时
  - 从runnext拿
  - runnext没有，从local拿
  - local都kong，从全局拿一半，然后放到local
  - 全局也是空，从其他拿P的local尾巴偷一半



6-17：58

## Netpoll

网络轮询负责管理网络FD相关的读写、就绪事



## Memory

当代码需要内存时，负责内存分配工作



## Garbage

当内存不再需要时，负责回收内存



# 内置数据结构

shimo.im/docs/C8pcgwVKvXVxJjpV

## channel

ubBuffered



buffered （循环数组buf+发送队列sendq）

写数据

![image-20211226215758528](C:\Users\zcw\AppData\Roaming\Typora\typora-user-images\image-20211226215758528.png)

- 判断channel是否为空
- 如果buf写满，打包sudog，挂到sendq链表上

读数据

## Timer 四叉堆


调整: 

Timerheap和GMP中的P绑定

去除唤醒goroutine: timerproc

检查:

检查timer到期在特殊函数checkTimers 中进行

检查timer操作移至调度循环中进行

工作窃取:

work-stealing中，会从其它P那里偷timer

兜底:

runtime.sysmon中会为timer未被触发(timeSleepUntil)兜底,启动新线程



## Map

![image-20211227203907453](/Users/zdns/Library/Application Support/typora-user-images/image-20211227203907453.png)

map添加



map扩容



map访问

缺陷：map没法缩容量



**频繁分配的场景用sync.Pool重用**



## Context

![image-20211227205938348](/Users/zdns/Library/Application Support/typora-user-images/image-20211227205938348.png)

在父子之间传递



https://mr-dai.github.io/mit-6824-lab1/

https://juejin.cn/post/6939447065357320206

https://siegelion.cn/2021/09/24/MIT%206.824-Lab%201/

http://guodong.plus/2020/1227-214432/

https://zou.cool/2018/11/27/mapreduce/



# Go的系统调用

syscall

- sysnb
  - RawSyscall
  - RawSyscall6
- sys
  - Syscall
  - Syscall6

# 内存分配与垃圾回收

## 分配器

虚拟内存布局



**Allocator**

- First-Fit

- Next-Fit

- Best-Fit

- Segregated-Fit

**Malloc**

- brk：调整prograbreak
- mmap

**Dangling Pointer**问题

## 内存分配

分配大小分类

- Tiny：size<16  && has no pointer nscan
- Small: has pointer(scan) || size >= 16b && size <= 32kb
- Large: size > 32KB

