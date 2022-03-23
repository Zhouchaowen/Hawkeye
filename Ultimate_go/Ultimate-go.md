**先写对，再按必要优化**

# 1.程序设计

- 延时
- 内存分配，垃圾回收
- 访问数据效率
- 算法效率

# 2.go语法

## 2-1变量

- 变量的类型：占用多大内存(分配读取多少字节)，这块内存怎样表示这个变量
- 优选考虑转化，而不是转型
- var关键字
- string 结构【指针|长度】

## 2-2.Struct

- 内存分配

```GO
// example represents a type with different fields.
type example struct {
	flag    bool
	counter int16
	pi      float32
}
// 内存对齐后是8字节 flag:2,counter:2,pi:4
```

- 内存对齐与填充(以结构体内最大属性内存占用的倍数填充)

- 具名类型和字面类型

```go
// example represents a type with different fields.
type example struct {
	flag    bool
	counter int16
	pi      float32
}

func main() {

	// Declare a variable of an anonymous type and init
	// using a struct literal.
	e := struct {
		flag    bool
		counter int16
		pi      float32
	}{
		flag:    true,
		counter: 10,
		pi:      3.141592,
	}

	// Create a value of type example.
	var ex example

	// Assign the value of the unnamed struct type
	// to the named struct type value.
	ex = e

	// Display the values.
	fmt.Printf("%+v\n", ex)
	fmt.Printf("%+v\n", e)
	fmt.Println("Flag", e.flag)
	fmt.Println("Counter", e.counter)
	fmt.Println("Pi", e.pi)
}
```

## 2-3.指针

**栈和内存帧**，每一行代码都会操作内存，goroutine只能对内存直接访问

指针是一个字面类型。***int 是个int类型 的 指针**

- go都是按值传递（所见即所得）
- 值语义
- 指针语义
- int和*int是不一样的类型

值传递保存程序简单性但会带来内存占用问题，指针传递可以只使用一份数据，但会导致程序复杂，破坏数据，引发bug

- **溢出分析**
  - 分析 函数返回结构体指针
  - 编译器无法确定 内存分配的大小
  - 每个gorouting栈大小为2k
    - 每次扩大25%

**绝对不要把刚构造好的值赋值给指针**

- 垃圾回收优化 pacing算法 （协作调度，**执行耗费cpu的运算一定要 执行函数调用**）
  - 把堆的尺寸压缩到最低（减少gc扫描标记的大小）
  - 让垃圾收集器以合理的步调运作，让程序应为stw的时间小于100微秒。
  - 利用多达25%的cpu处理能力。

## 2-4.常量

kind promtion 

- 带类型的常量 

```go
const int 
```



- 不带类型的常量，精确的256位



# 3.数据结构

**面向数据编程**

## 3-1.数组

- slice和链表
  - 矩阵行列遍历和链表遍历
  - 4级缓存,**让数据提前出现到离cpu更近的地方**，可预知的方式访问数据
    - L1
    - L2
    - L3  cache 8M
    - 主内存
  - 值语义和指针语言下 for 循环和 for range循环的区别

## 3-2.切片

 



## 3-3.映射



# 4.解耦

## 4-1.方法

使用方法只是特殊情况

原则：

1**.内置类型：字符串，数值，bool应该用值语义（内置类型就应该使用值语义）**

2.**应用类型：切片，映射，channle也应该用值语义（除了需要访问他们内部值的情况外（拆分和解码））**

3.结构体：优先指针语义，在值语义。





## 4-2函数

数据什么时候应该表现解耦（多态）

**数据的行为**

**接口传递：需要传递一个包含接口所有行为的数据。**

```go
type reader interface{
	read()
}

func retriever(r reader) error{
	r.read()
}
```

