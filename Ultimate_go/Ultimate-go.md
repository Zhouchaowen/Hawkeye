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





## 4-2.函数

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

## 4-3.组合

归组: 关注它能做什么，而不同它属于什么

标准库为什么要定义：type Duration int64

**解耦要依据行为**

1.从具体数据出发

2.分层：原始层，低级层，高级层

3.分部解决问题：第一步，第二步，第三步，重构.....

```golang
type User struct{
    name string
    email string
    age int
}

func (u *user) SendEmail(){} // 这个最不好
func SendEmail(u *user){}
func SendEmail(name string,email string,age int){} // 这个最好
```

依赖的是接口并不是具体的值

```golang
// 例子1
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// Sample program demonstrating decoupling with interfaces.
package main

import (
	"errors"
	"fmt"
	"io"
	"math/rand"
	"time"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}

// =============================================================================

// Data is the structure of the data we are copying.
type Data struct {
	Line string
}

// =============================================================================

// Puller declares behavior for pulling data.
type Puller interface {
	Pull(d *Data) error
}

// Storer declares behavior for storing data.
type Storer interface {
	Store(d *Data) error
}

// =============================================================================

// Xenia is a system we need to pull data from.
type Xenia struct {
	Host    string
	Timeout time.Duration
}

// Pull knows how to pull data out of Xenia.
func (*Xenia) Pull(d *Data) error {
	switch rand.Intn(10) {
	case 1, 9:
		return io.EOF

	case 5:
		return errors.New("Error reading data from Xenia")

	default:
		d.Line = "Data"
		fmt.Println("In:", d.Line)
		return nil
	}
}

// Pillar is a system we need to store data into.
type Pillar struct {
	Host    string
	Timeout time.Duration
}

// Store knows how to store data into Pillar.
func (*Pillar) Store(d *Data) error {
	fmt.Println("Out:", d.Line)
	return nil
}

// =============================================================================

// System wraps Xenia and Pillar together into a single system.
type System struct {
	Xenia
	Pillar
}

// =============================================================================

// pull knows how to pull bulks of data from any Puller.
func pull(p Puller, data []Data) (int, error) {
	for i := range data {
		if err := p.Pull(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// store knows how to store bulks of data from any Storer.
func store(s Storer, data []Data) (int, error) {
	for i := range data {
		if err := s.Store(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// Copy knows how to pull and store data from the System.
func Copy(sys *System, batch int) error {
	data := make([]Data, batch)

	for {
		i, err := pull(&sys.Xenia, data)
		if i > 0 {
			if _, err := store(&sys.Pillar, data[:i]); err != nil {
				return err
			}
		}

		if err != nil {
			return err
		}
	}
}

// =============================================================================

func main() {
	sys := System{
		Xenia: Xenia{
			Host:    "localhost:8000",
			Timeout: time.Second,
		},
		Pillar: Pillar{
			Host:    "localhost:9000",
			Timeout: time.Second,
		},
	}

	if err := Copy(&sys, 3); err != io.EOF {
		fmt.Println(err)
	}
}

```

```golang
// 优化例子1
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// Sample program demonstrating interface composition.
package main

import (
	"errors"
	"fmt"
	"io"
	"math/rand"
	"time"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}

// =============================================================================

// Data is the structure of the data we are copying.
type Data struct {
	Line string
}

// =============================================================================

// Puller declares behavior for pulling data.
type Puller interface {
	Pull(d *Data) error
}

// Storer declares behavior for storing data.
type Storer interface {
	Store(d *Data) error
}

// PullStorer declares behavior for both pulling and storing.
type PullStorer interface {
	Puller
	Storer
}

// =============================================================================

// Xenia is a system we need to pull data from.
type Xenia struct {
	Host    string
	Timeout time.Duration
}

// Pull knows how to pull data out of Xenia.
func (*Xenia) Pull(d *Data) error {
	switch rand.Intn(10) {
	case 1, 9:
		return io.EOF

	case 5:
		return errors.New("Error reading data from Xenia")

	default:
		d.Line = "Data"
		fmt.Println("In:", d.Line)
		return nil
	}
}

// Pillar is a system we need to store data into.
type Pillar struct {
	Host    string
	Timeout time.Duration
}

// Store knows how to store data into Pillar.
func (*Pillar) Store(d *Data) error {
	fmt.Println("Out:", d.Line)
	return nil
}

// =============================================================================

// System wraps Xenia and Pillar together into a single system.
type System struct {
	Xenia
	Pillar
}

// =============================================================================

// pull knows how to pull bulks of data from any Puller.
func pull(p Puller, data []Data) (int, error) {
	for i := range data {
		if err := p.Pull(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// store knows how to store bulks of data from any Storer.
func store(s Storer, data []Data) (int, error) {
	for i := range data {
		if err := s.Store(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// Copy knows how to pull and store data from any System.
func Copy(ps PullStorer, batch int) error {
	data := make([]Data, batch)

	for {
		i, err := pull(ps, data)
		if i > 0 {
			if _, err := store(ps, data[:i]); err != nil {
				return err
			}
		}

		if err != nil {
			return err
		}
	}
}

// =============================================================================

func main() {
	sys := System{
		Xenia: Xenia{
			Host:    "localhost:8000",
			Timeout: time.Second,
		},
		Pillar: Pillar{
			Host:    "localhost:9000",
			Timeout: time.Second,
		},
	}

	if err := Copy(&sys, 3); err != io.EOF {
		fmt.Println(err)
	}
}
```



```golang
// 优化2
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// Sample program demonstrating decoupling with interface composition.
package main

import (
	"errors"
	"fmt"
	"io"
	"math/rand"
	"time"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}

// =============================================================================

// Data is the structure of the data we are copying.
type Data struct {
	Line string
}

// =============================================================================

// Puller declares behavior for pulling data.
type Puller interface {
	Pull(d *Data) error
}

// Storer declares behavior for storing data.
type Storer interface {
	Store(d *Data) error
}

// PullStorer declares behavior for both pulling and storing.
type PullStorer interface {
	Puller
	Storer
}

// =============================================================================

// Xenia is a system we need to pull data from.
type Xenia struct {
	Host    string
	Timeout time.Duration
}

// Pull knows how to pull data out of Xenia.
func (*Xenia) Pull(d *Data) error {
	switch rand.Intn(10) {
	case 1, 9:
		return io.EOF

	case 5:
		return errors.New("Error reading data from Xenia")

	default:
		d.Line = "Data"
		fmt.Println("In:", d.Line)
		return nil
	}
}

// Pillar is a system we need to store data into.
type Pillar struct {
	Host    string
	Timeout time.Duration
}

// Store knows how to store data into Pillar.
func (*Pillar) Store(d *Data) error {
	fmt.Println("Out:", d.Line)
	return nil
}

// =============================================================================

// System wraps Pullers and Stores together into a single system.
type System struct {
	Puller
	Storer
}

// =============================================================================

// pull knows how to pull bulks of data from any Puller.
func pull(p Puller, data []Data) (int, error) {
	for i := range data {
		if err := p.Pull(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// store knows how to store bulks of data from any Storer.
func store(s Storer, data []Data) (int, error) {
	for i := range data {
		if err := s.Store(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// Copy knows how to pull and store data from any System.
func Copy(ps PullStorer, batch int) error {
	data := make([]Data, batch)

	for {
		i, err := pull(ps, data)
		if i > 0 {
			if _, err := store(ps, data[:i]); err != nil {
				return err
			}
		}

		if err != nil {
			return err
		}
	}
}

// =============================================================================

func main() {
	sys := System{
		Puller: &Xenia{
			Host:    "localhost:8000",
			Timeout: time.Second,
		},
		Storer: &Pillar{
			Host:    "localhost:9000",
			Timeout: time.Second,
		},
	}

	if err := Copy(&sys, 3); err != io.EOF {
		fmt.Println(err)
	}
}

```



```golang
// 优化3
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// Sample program demonstrating removing interface pollution.
package main

import (
	"errors"
	"fmt"
	"io"
	"math/rand"
	"time"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}

// =============================================================================

// Data is the structure of the data we are copying.
type Data struct {
	Line string
}

// =============================================================================

// Puller declares behavior for pulling data.
type Puller interface {
	Pull(d *Data) error
}

// Storer declares behavior for storing data.
type Storer interface {
	Store(d *Data) error
}

// =============================================================================

// Xenia is a system we need to pull data from.
type Xenia struct {
	Host    string
	Timeout time.Duration
}

// Pull knows how to pull data out of Xenia.
func (*Xenia) Pull(d *Data) error {
	switch rand.Intn(10) {
	case 1, 9:
		return io.EOF

	case 5:
		return errors.New("Error reading data from Xenia")

	default:
		d.Line = "Data"
		fmt.Println("In:", d.Line)
		return nil
	}
}

// Pillar is a system we need to store data into.
type Pillar struct {
	Host    string
	Timeout time.Duration
}

// Store knows how to store data into Pillar.
func (*Pillar) Store(d *Data) error {
	fmt.Println("Out:", d.Line)
	return nil
}

// =============================================================================

// System wraps Pullers and Stores together into a single system.
type System struct {
	Puller
	Storer
}

// =============================================================================

// pull knows how to pull bulks of data from any Puller.
func pull(p Puller, data []Data) (int, error) {
	for i := range data {
		if err := p.Pull(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// store knows how to store bulks of data from any Storer.
func store(s Storer, data []Data) (int, error) {
	for i := range data {
		if err := s.Store(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// Copy knows how to pull and store data from any System.
func Copy(sys *System, batch int) error {
	data := make([]Data, batch)

	for {
		i, err := pull(sys, data)
		if i > 0 {
			if _, err := store(sys, data[:i]); err != nil {
				return err
			}
		}

		if err != nil {
			return err
		}
	}
}

// =============================================================================

func main() {
	sys := System{
		Puller: &Xenia{
			Host:    "localhost:8000",
			Timeout: time.Second,
		},
		Storer: &Pillar{
			Host:    "localhost:9000",
			Timeout: time.Second,
		},
	}

	if err := Copy(&sys, 3); err != io.EOF {
		fmt.Println(err)
	}
}
```



```golang
// 优化4
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// Sample program demonstrating being more precise with API design.
package main

import (
	"errors"
	"fmt"
	"io"
	"math/rand"
	"time"
)

func init() {
	rand.Seed(time.Now().UnixNano())
}

// =============================================================================

// Data is the structure of the data we are copying.
type Data struct {
	Line string
}

// =============================================================================

// Puller declares behavior for pulling data.
type Puller interface {
	Pull(d *Data) error
}

// Storer declares behavior for storing data.
type Storer interface {
	Store(d *Data) error
}

// =============================================================================

// Xenia is a system we need to pull data from.
type Xenia struct {
	Host    string
	Timeout time.Duration
}

// Pull knows how to pull data out of Xenia.
func (*Xenia) Pull(d *Data) error {
	switch rand.Intn(10) {
	case 1, 9:
		return io.EOF

	case 5:
		return errors.New("Error reading data from Xenia")

	default:
		d.Line = "Data"
		fmt.Println("In:", d.Line)
		return nil
	}
}

// Pillar is a system we need to store data into.
type Pillar struct {
	Host    string
	Timeout time.Duration
}

// Store knows how to store data into Pillar.
func (*Pillar) Store(d *Data) error {
	fmt.Println("Out:", d.Line)
	return nil
}

// =============================================================================

// pull knows how to pull bulks of data from any Puller.
func pull(p Puller, data []Data) (int, error) {
	for i := range data {
		if err := p.Pull(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// store knows how to store bulks of data from any Storer.
func store(s Storer, data []Data) (int, error) {
	for i := range data {
		if err := s.Store(&data[i]); err != nil {
			return i, err
		}
	}

	return len(data), nil
}

// Copy knows how to pull and store data from any System.
func Copy(p Puller, s Storer, batch int) error {
	data := make([]Data, batch)

	for {
		i, err := pull(p, data)
		if i > 0 {
			if _, err := store(s, data[:i]); err != nil {
				return err
			}
		}

		if err != nil {
			return err
		}
	}
}

// =============================================================================

func main() {
	x := Xenia{
		Host:    "localhost:8000",
		Timeout: time.Second,
	}

	p := Pillar{
		Host:    "localhost:9000",
		Timeout: time.Second,
	}

	if err := Copy(&x, &p, 3); err != io.EOF {
		fmt.Println(err)
	}
}
```

**错误示例**

```golang
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// This is an example that creates interface pollution
// by improperly using an interface when one is not needed.
package main

// Server defines a contract for tcp servers.
type Server interface {
	Start() error
	Stop() error
	Wait() error
}

// server is our Server implementation.
type server struct {
	host string

	// PRETEND THERE ARE MORE FIELDS.
}

// NewServer returns an interface value of type Server
// with a server implementation.
func NewServer(host string) Server {

	// SMELL - Storing an unexported type pointer in the interface.
	return &server{host}
}

// Start allows the server to begin to accept requests.
func (s *server) Start() error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}

// Stop shuts the server down.
func (s *server) Stop() error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}

// Wait prevents the server from accepting new connections.
func (s *server) Wait() error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}

func main() {

	// Create a new Server.
	srv := NewServer("localhost")

	// Use the API.
	srv.Start()
	srv.Stop()
	srv.Wait()
}

// =============================================================================

// NOTES:

// Smells:
//  * The package declares an interface that matches the entire API of its own concrete type.
//  * The interface is exported but the concrete type is unexported.
//  * The factory function returns the interface value with the unexported concrete type value inside.
//  * The interface can be removed and nothing changes for the user of the API.
//  * The interface is not decoupling the API from change.
```



```golang
// 修改后
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// This is an example that removes the interface pollution by
// removing the interface and using the concrete type directly.
package main

// Server is our Server implementation.
type Server struct {
	host string

	// PRETEND THERE ARE MORE FIELDS.
}

// NewServer returns an interface value of type Server
// with a server implementation.
func NewServer(host string) *Server {

	// SMELL - Storing an unexported type pointer in the interface.
	return &Server{host}
}

// Start allows the server to begin to accept requests.
func (s *Server) Start() error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}

// Stop shuts the server down.
func (s *Server) Stop() error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}

// Wait prevents the server from accepting new connections.
func (s *Server) Wait() error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}

func main() {

	// Create a new Server.
	srv := NewServer("localhost")

	// Use the API.
	srv.Start()
	srv.Stop()
	srv.Wait()
}

// =============================================================================

// NOTES:

// Here are some guidelines around interface pollution:
// * Use an interface:
//      * When users of the API need to provide an implementation detail.
//      * When API’s have multiple implementations that need to be maintained.
//      * When parts of the API that can change have been identified and require decoupling.
// * Question an interface:
//      * When its only purpose is for writing testable API’s (write usable API’s first).
//      * When it’s not providing support for the API to decouple from change.
//      * When it's not clear how the interface makes the code better.
```

**约定大于配置**

```golang
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// Package pubsub simulates a package that provides publication/subscription
// type services.
package pubsub

// PubSub provides access to a queue system.
type PubSub struct {
	host string

	// PRETEND THERE ARE MORE FIELDS.
}

// New creates a pubsub value for use.
func New(host string) *PubSub {
	ps := PubSub{
		host: host,
	}

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.

	return &ps
}

// Publish sends the data for the specified key.
func (ps *PubSub) Publish(key string, v interface{}) error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}

// Subscribe sets up an request to receive messages for the specified key.
func (ps *PubSub) Subscribe(key string) error {

	// PRETEND THERE IS A SPECIFIC IMPLEMENTATION.
	return nil
}
```

```golang
// All material is licensed under the Apache License Version 2.0, January 2004
// http://www.apache.org/licenses/LICENSE-2.0

// Sample program to show how you can personally mock concrete types when
// you need to for your own packages or tests.
package main

import (
	"github.com/ardanlabs/gotraining/topics/go/design/composition/mocking/example1/pubsub"
)

// publisher is an interface to allow this package to mock the pubsub
// package support.
type publisher interface {
	Publish(key string, v interface{}) error
	Subscribe(key string) error
}

// mock is a concrete type to help support the mocking of the pubsub package.
type mock struct{}

// Publish implements the publisher interface for the mock.
func (m *mock) Publish(key string, v interface{}) error {

	// ADD YOUR MOCK FOR THE PUBLISH CALL.
	return nil
}

// Subscribe implements the publisher interface for the mock.
func (m *mock) Subscribe(key string) error {

	// ADD YOUR MOCK FOR THE SUBSCRIBE CALL.
	return nil
}

func main() {

	// Create a slice of publisher interface values. Assign
	// the address of a pubsub.PubSub value and the address of
	// a mock value.
	pubs := []publisher{
		pubsub.New("localhost"),
		&mock{},
	}

	// Range over the interface value to see how the publisher
	// interface provides the level of decoupling the user needs.
	// The pubsub package did not need to provide the interface type.
	for _, p := range pubs {
		p.Publish("key", "value")
		p.Subscribe("key")
	}
}
```

# 5.错误处理







