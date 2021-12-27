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

## Timer 死叉堆


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

9-54：45

## Map

