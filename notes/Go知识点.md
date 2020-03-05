
  
### 1.struct能不能比较

不同类型的struct不能作比较,相同类型的struct只有全部成员都是可以比较的(slice, map都是不可比较的)，才可用`==`比较。

`reflect.DeepEqual()`不但可以比较结构体，像切片、数组和字典等都可以比较。 以下两种情况:  

1. 指针1 == 指针2 结果为true

2. 指针1 指针2 所指向的值深度相等

满足其中的一条即 DeepEqual 结果为true

### 2.类型断言

1. 如果断言的类型T是一个具体类型，然后类型断言检查x的动态类型是否和T相同。如果这个检查成功了，类型断言的结果是x的动态值，当然它的类型是T  

2. 断言的类型T是一个接口类型，然后类型断言检查是否x的动态类型满足T。如果这个检查成功了，动态值没有获取到；这个结果仍然是一个有相同类型和值部分的接口值
，但是结果有类型T。换句话说，对一个接口类型的类型断言改变了类型的表述方式，改变了可以获取的方法集合（通常更大），但是它保护了接口值内部的动态类型和值的部分。


### 3.判断interface{}的类型
```
1.类型断言: v, ok := x.(int)
2.反射: reflect.TypeOf(x)
```


### 5.channel的注意点

1. 往一个已经被close的channel中发送数据会导致run-time panic

2. 从一个被close的channel中接收数据不会被阻塞，而是立即返回，接收完已发送的数据后会返回元素类型的零值

3. 重复close会panic，可以用recover捕获

4. 往nil channel中发送和接收数据都会一致被阻塞着

5. 有缓存的channle， 只有buffer满了后 send才会阻塞， 只有缓存空了后receive才会阻塞

6. x, ok := <-ch检查关闭

7. range <-ch产生的迭代值为Channel中发送的值，它会一直迭代直到channel被close

对于从无缓冲Channel进行的接收，发生在对该Channel进行的发送完成之前

对于带缓冲的Channel，对于Channel的第K个接收完成操作发生在第K+C个发送操作完成之前，其中C是Channel的缓存大小


### 6.Context的作用

通常被译作上下文，它是一个比较抽象的概念，其本质，是【上下上下】存在上下层的传递，上会把内容传递给下。在Go语言中，程序单元也就指的是Goroutine
主要用来管理Goroutine的生命周期


### 7.主协程如何等其余协程完再操作  

1. sync.WaitGroup对象和它的的Add(),Done(),Wait()
2. channel通信
3. context


## 溢出

```go
a := int8(127)
b := a << 1    //结果: -2
c := int8(127) << 1  //编译报错: constant 254 overflows int8
fmt.Println("b:", b)
fmt.Println("c:", c)
```

## 切片和数组

slice底层数据结构是由数组、len、cap组成，所以，在使用append()扩容时，会查看数组后面有没有连续内存块，有就在后面添加，没有就重新生成一个大的数组，新数组容量是原来的2倍(当前切片容量等于 1024并用完之后, append操作 就会每次增加 25% 的容量，直到新容量大于期望容量)

注意:
- nil或容量为0的slice可以append，但不能用s[0]=10这样赋值
- 如果a为容量为6的slice,a[6]不越界，等于[]
- 相比于依次对元素进行拷贝，copy()这种方式能够提供更好的性能，使用 memmove 对内存成块进行拷贝，但是这个操作还是会占用非常多的资源，在大切片上执行拷贝操作时一定要注意性能影响

### 区别:  
数组: 
- 数组是一个由固定长度的特定类型元素组成的序列
- 数组的长度是数组类型的一个组成部分，因此[3]int和[4]int是两种不同的数组类型
- 数组作为函数的参数，那么实际传递的参数是一份数组的拷贝,除非显式传入指针

slice:  
- 一个slice由三个部分构成：指针、长度和容量。指针指向第一个slice元素对应的底层数组元素的地址
- 修改slice会修改底层的数组元素
- 扩容会引用一个新的底层数组

```
var s []int    // len(s) == 0, s == nil
s = nil        // len(s) == 0, s == nil
s = []int(nil) // len(s) == 0, s == nil
s = []int{}    // len(s) == 0, s != nil
```
## 哈希表

哈希表要关键的两点：一是选择合理的哈希函数，保证分布均匀，第二如何解决哈希碰撞  
解决哈希碰撞:  
- **开放寻址法**:用数组实现，装载因子过大，线性探测的平均用时就会逐渐增加，这会同时影响哈希表的读写性能
- **拉链法**: 实现拉链法一般会使用数组加上链表，不过有一些语言会在拉链法的哈希中引入红黑树以优化性能，拉链法会使用链表数组作为哈希底层的数据结构。在一般情况下使用拉链法的哈希表装载因子都不会超过 1，当哈希表的装载因子较大时就会触发哈希的扩容，创建更多的桶来存储哈希中的元素，保证性能不会出现严重的下降。

Go 语言使用拉链法来解决哈希碰撞的问题实现了哈希表，它的访问、写入和删除等操作都在编译期间转换成了运行时的函数或者方法。

哈希在每一个桶中存储键对应哈希的前 8 位，当对哈希进行操作时，这些 tophash 就成为了一级缓存帮助哈希快速遍历桶中元素，每一个桶都只能存储 8 个键值对，一旦当前哈希的某个桶超出 8 个，新的键值对就会被存储到哈希的溢出桶中。

随着键值对数量的增加，溢出桶的数量和哈希的装载因子也会逐渐升高，超过一定范围就会触发扩容，扩容会将桶的数量翻倍，元素再分配的过程也是在调用写操作时增量进行的，不会造成性能的瞬时巨大抖动。
 
go的map： 
- nil的map不能用插入新成员
- m[key]不存在会返回value类型的零值
- delete(m, key)删除不存在key也可以，无返回
- map不能顺序读取，是因为他是无序的，想要有序读取，可以把key放入切片，对切片进行排序，遍历切片，通过key取值map的值


## 函数

Go 通过栈传递函数的参数和返回值，在调用函数之前会在栈上为返回值分配合适的内存空间，随后将入参从右到左按顺序压栈并拷贝参数，返回值会被存储到调用方预留好的栈空间上，我们可以简单总结出以下几条规则:
- 通过堆栈传递参数，入栈的顺序是从右到左；
- 函数返回值通过堆栈传递并由调用者预先分配内存空间；
- 调用函数时都是传值，接收方会对入参进行复制再计算；

## 接口

当我们使用指针实现接口时，只有指针类型的变量才会实现该接口；当我们使用结构体实现接口时，指针类型和结构体类型都会实现该接口

interface{}
```go
package main

import "fmt"

type TestStruct struct{}

func NilOrNot(v interface{}) bool {
	return v == nil
}

func main() {
	var s *TestStruct
	fmt.Println(s == nil)    // #=> true
	fmt.Println(NilOrNot(s)) // #=> false
}

```
原因:   
调用 NilOrNot 函数时发生了隐式的类型转换，除了向方法传入参数之外，变量的赋值也会触发隐式类型转换。在类型转换时，*TestStruct 类型会转换成 interface{} 类型，转换后的变量不仅包含转换前的变量，还包含变量的类型信息 TestStruct，所以转换后的变量与 nil 不相等。


动态派发:

动态派发（Dynamic dispatch）是在运行期间选择具体多态操作（方法或者函数）执行的过程，它是一种在面向对象语言中常见的特性。Go 语言虽然不是严格意义上的面向对象语言，但是接口的引入为它带来了动态派发这一特性，调用接口类型的方法时，如果编译期间不能确认接口的类型，Go 语言会在运行期间决定具体调用该方法的哪个实现。

使用结构体来实现接口带来的开销会大于使用指针实现，而动态派发在结构体上的表现非常差，这也提醒我们应当尽量避免使用结构体类型实现接口

使用结构体带来的巨大性能差异不只是接口带来的问题，带来性能问题主要因为 Go 语言在函数调用时是传值的，动态派发的过程只是放大了参数拷贝带来的影响

## 反射


运行时反射是程序在运行期间检查其自身结构的一种方式。反射带来的灵活性是一把双刃剑，反射作为一种元编程方式可以减少重复代码，但是过量的使用反射会使我们的程序逻辑变得难以理解并且运行缓慢。

Go 语言反射的三大法则3，其中包括：  
- 从 interface{} 变量可以反射出反射对象；
- 从反射对象可以获取 interface{} 变量；
- 要修改反射对象，其值必须可设置；

反射实现了运行时的反射能力，能够让程序操作不同类型的对象。可以动态修改变量、判断类型是否实现了某些接口以及动态调用方法等。


## for range

对于所有的 range 循环，Go 语言都会在编译期将原切片或者数组赋值给一个新的变量 ha，在赋值的过程中就发生了拷贝，所以我们遍历的切片已经不是原始的切片变量了。  


错误1:  
```go
func main() {
	arr := []int{1, 2, 3}
	newArr := []*int{}
	for _, v := range arr {
		newArr = append(newArr, &v)
	}
	for _, v := range newArr {
		fmt.Println(*v)
	}
}

$ go run main.go
3 3 3
```
上面正确的做法是: 使用 &arr[i] 替代 &v  
原因： 循环中使用的这个变量 v 会在每一次迭代被重新赋值，在赋值时也发生了拷贝   

错误2:  

```go
for _, dir := range tempDirs() {
	// dir := dir // declares inner dir, initialized to outer dir
    os.MkdirAll(dir, 0755)
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dir) // NOTE: incorrect!
    })
}
```
这个问题不仅存在基于range的循环，在下面的例子中，匿名函数对循环变量i的使用也存在同样的问题

错误3：  
```go
var rmdirs []func()
dirs := tempDirs()
for i := 0; i < len(dirs); i++ {
    os.MkdirAll(dirs[i], 0755) // OK
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dirs[i]) // NOTE: incorrect!
    })
}
```

遍历字符串的过程与数组、切片和哈希表非常相似，只是在遍历时会获取字符串中索引对应的字节并将字节转换成 rune。我们在遍历字符串时拿到的值都是 rune 类型的变量。


## select

select {} 的空语句会直接阻塞当前的 Goroutine，让出当前 Goroutine 对处理器的使用权，该 Goroutine 也会进入永久休眠的状态也没有办法被其他的 Goroutine 唤醒。

非空的select{} 等到 select 对应的一些 Channel 准备好之后，当前 Goroutine 就会被调度器唤醒。


## defer

代码1:
```go
type Test struct {
    value int
}

func (t Test) print() {
    println(t.value)
}

func main() {
	test := Test{}
	defer test.print()
	test.value += 1
}

$ go run main.go
0
```
这其实表明当 defer 调用时其实会对函数中引用的外部参数进行拷贝，所以 test.value += 1 操作并没有修改被 defer 捕获的 test 结构体，不过如果我们修改 print 函数签名的话，把结构体接收者换成指针接收者，其实结果就会稍有不同。

规则:
1. 当defer被声明时，其参数就会被实时解析
2. defer执行顺序为先进后出
3. defer可以读取有名返回值

用法:
1. 释放占用的资源
2. 捕捉处理异常
3. 输出日志 等收尾工作

注意:
[Golang中defer、return、返回值之间执行顺序的坑](https://studygolang.com/articles/4809)

## panic 和 recover

`panic` 能够改变程序的控制流，当一个函数调用执行 panic 时，它会立刻停止执行函数中其他的代码，而是会运行其中的 defer 函数，执行成功后会返回到调用方。

对于上层调用方来说，调用导致 panic 的函数其实与直接调用 panic 类似，所以也会执行所有的 defer 函数并返回到它的调用方，这个过程会一直进行直到当前 Goroutine 的调用栈中不包含任何的函数，这时整个程序才会崩溃，这个『恐慌过程』不仅会被显式的调用触发，还会由于运行期间发生错误而触发。


正确：
```go
func main() {
	defer func() {
		recover()
	}()
	panic("ooo")
}
```
错误，照样panic:
```go
func main() {
	defer recover()
	panic("ooo")
}
```

原因: Recover is only useful inside deferred functions

现象:  
```go
func main() {
	defer println("in main")
	go func() {
		defer println("in goroutine")
		panic("")
	}()

	time.Sleep(1 * time.Second)
}

// in goroutine
// panic:
// ...
```
原因:  Go 语言在发生 panic 时只会执行当前协程中的 defer 函数


## channel

### 发送

我们在这里可以简单梳理和总结一下使用 ch <- i 表达式向 Channel 发送数据时遇到的几种情况：

1. 如果当前 Channel 的 recvq 上存在已经被阻塞的 Goroutine，那么会直接将数据发送给当前的 Goroutine 并将其设置成下一个运行的协程；  
2. 如果 Channel 存在缓冲区并且其中还有空闲的容量，我们就会直接将数据直接存储到当前缓冲区 sendx 所在的位置上；
3. 如果都不满足上面的两种情况，就会创建一个 sudog 结构并加入 Channel 的 sendq 队列并更新到 Goroutine 的 waiting 字段上，同时当前的 Goroutine 就会陷入阻塞等待其他的协程向 Channel 接收数据，一旦有其它的协程向 Channel 接收数据时就会唤醒当前的 Goroutine；


发送数据的过程中包含几个会触发 Goroutine 调度的时机，首先是发送数据时发现 Channel 上存在等待接收数据的 Goroutine，这是会立刻设置处理器的 runnext 属性，但是并不会立刻触发调度，第二个时机是发送数据时并没有找到接收方并且缓冲区已经满了，这时就会将自己加入 Channel 的 sendq 队列并立刻调用 goparkunlock 触发 Goroutine 的调度让出处理器的使用权。

### 接收

我们简单梳理一下从 Channel 中接收数据时的几种情况：

1. 如果 Channel 是空的，那么就会直接调用 gopark 挂起当前的 Goroutine；
2. 如果 Channel 已经关闭并且缓冲区没有任何数据，chanrecv 函数就会直接返回；
3. 如果 Channel 上的 sendq 队列中存在挂起的 Goroutine，就会将recvx 索引所在的数据拷贝到接收变量所在的内存空间上并将 sendq 队列中 Goroutine 的数据拷贝到缓冲区中；
4. 如果 Channel 的缓冲区中包含数据就会直接从 recvx 所在的索引上进行读取；
5. 在默认情况下会直接挂起当前的 Goroutine，将 sudog 结构加入 recvq 队列并更新 Goroutine 的 waiting 属性，最后陷入休眠等待调度器的唤醒；

在从管道中接收数据的过程中，其实会在两个时间点触发 Goroutine 的调度，首先空的 Channel 意味着永远接收不到消息，那么就会直接挂起当前 Goroutine，第二个时间点是缓冲区中不存在数据，在这时也会直接挂起当前的 Goroutine 等待发送方发送数据。

### 10.new和make的区别

make 关键字的主要作用是创建切片、哈希表和 Channel 等内置的数据结构，而 new 的主要作用是为类型申请一片内存空间，并返回指向这片内存的指针。

下面代码片段中的两种不同初始化方法其实是等价的，它们都会创建一个指向 int 零值的指针
```go
i := new(int)

var v int
i := &v
```


10.如何（优雅的）比较两个未知结构的json

11.原子操作
```
原子操作由底层硬件支持，而锁则由操作系统提供的API实现。若实现相同的功能，前者通常会更有效率
```

11.sync包的常用结构

1. 互斥锁 Mutex   
2. 读写互斥锁 RWMutex  
3. 单次执行 Once  
4. 临时对象池子 Pool  
5. Waitgroup  
6. 条件变量 Cond
7. 并发安全 Map

11.sync.Map

核心思想是用空间换时间，用两个map来存储数据，read和dirty，read支持原子操作，可以看作是dirty 的cache，dirty是更底层的数据存储层。

4种操作：读key、增加key、更新key、删除key的基本流程：

读key：先到read中读取，如果有则直接返回结果，如果没有或者是被删除（有特殊value值可以判断），则到dirty加锁中读取，如果有返回结果并更新miss数  
增加key：直接增加到dirty中  
更新key：先到read中看看有没有，如果有直接更新key，如果没有则到dirty中更新  
删除key：先到read中看看有没有，如果有则直接更新为nil，如果没有则到dirty中直接删除  

read的替换：当read多次都没有命中数据，达到阈值，表示这个cache命中率太低，这时直接将整个read用dirty替换掉，然后dirty又重新置为nil，下一次再添加一个新key的时候，会触发一次read到dirty的复制，这样二者又保持了一致。

虽然read和dirty有冗余，但这些map的value数据是通过指针指向同一个数据，所以尽管实际的value会很大，但是冗余的空间占用还是有限的。

总结，如果对map的读操作远远多于写操作（写操作包括新增和删除key），那么sync.Map是很合适，能够大大提升性能




12.[sync.Pool的用途](https://studygolang.com/articles/3506)
```
重用对象，减少内存的分配，从而减轻gc的压力
缓存对象数量是没有限制(只受限于内存),对象Get()出来就没有了
pool包在init的时候注册了一个poolCleanup函数，它会清除所有的pool里面的所有缓存的对象;
该函数注册进去之后会在每次gc之前都会调用，因此sync.Pool 缓存的期限只是两次gc之间这段时间
```

13.go的锁是怎么实现的
```

```

14.go只有值传递

13.goroutine泄露
```
原因:
1.比如生产者和消费者的有各自的goroutine，消费者goroutine报错退出了，生产者卡在往 channel 发送数据这一步
2.ctx没有cancel
3.没有正确使用time.Timer

package main

import (
	"fmt"
	"time"
)

//Golang的Timer在源码中，实现的方式是以一个小顶堆来维护所有的Timer集合。接着启动一个独立的goroutine，
//循环从小顶堆中的检测最近一个到期的Timer的到期时间，接着它睡眠到最近一个定时器到期的时间。
//最后会执行开始时设定的回调函数。Timer到期之后，会被Golang的runtime从小项堆中删除，并等待GC回收资源
func main() {
	timer := time.NewTimer(3 * time.Second)

	go func() {
		<-timer.C
		fmt.Println("Timer has expired.")
	}()
    
	timer.Stop() //错误做法:Timer自带的channel并没有关闭，而且这个Timer已经被删除了,如果close它才会打印
	//timer.Reset(0 * time.Second) //正确做法:给Timer一个0秒的超时时间，让Timer立刻过期
	time.Sleep(60 * time.Second)
}


// 实时查看goroutine的个数
go func() {
        http.HandleFunc("/goroutines", func(w http.ResponseWriter, r *http.Request) {
            num := strconv.FormatInt(int64(runtime.NumGoroutine()), 10)
            w.Write([]byte(num))
        });
        http.ListenAndServe("localhost:6060", nil)
        glog.Info("goroutine stats and pprof listen on 6060")
    }()

```

14.用channel实现一个定时器
```
c := make(chan int)
go func() {
	time.Sleep(2 * time.Second)
	c <- 1
}()
<-c
fmt.Println("exit")
```

14.循环变量的作用域
```
//d中存储的值等于最后一次迭代的值
for _, d := range tempDirs() {
    dir := d // NOTE: necessary!
    os.MkdirAll(dir, 0755) // creates parent directories too
    rmdirs = append(rmdirs, func() {
        os.RemoveAll(dir)
    })
}
```
14.client如何实现长连接
```go
// net/http/client.go
type Client struct {
	// If nil, DefaultTransport is used.
	Transport RoundTripper
	CheckRedirect func(req *Request, via []*Request) error
	Jar CookieJar
	Timeout time.Duration
}

// net/http/transport.go
// DefaultTransport is the default implementation of Transport and is
// used by DefaultClient. It establishes network connections as needed
// and caches them for reuse by subsequent calls. It uses HTTP proxies
// as directed by the $HTTP_PROXY and $NO_PROXY (or $http_proxy and
// $no_proxy) environment variables.
var DefaultTransport RoundTripper = &Transport{
	Proxy: ProxyFromEnvironment,
	DialContext: (&net.Dialer{
		Timeout:   30 * time.Second,
		KeepAlive: 30 * time.Second,
		DualStack: true,
	}).DialContext,
	MaxIdleConns:          100,
	IdleConnTimeout:       90 * time.Second,
	TLSHandshakeTimeout:   10 * time.Second,
	ExpectContinueTimeout: 1 * time.Second,
	// DisableKeepAlives, if true, prevents re-use of TCP connections
	// default false
	DisableKeepAlives: false
}

const DefaultMaxIdleConnsPerHost = 2

```


15.golang实现一个set
```go
//map的key必须是支持==比较运算符的数据类型，不建议用浮点数

package main

import (
	"fmt"
	"sort"
	"sync"
)

type Set struct {
	m map[string]bool
	mu *sync.RWMutex
}

func NewSet() *Set {
	return &Set{
		m: make(map[string]bool),
		mu: new(sync.RWMutex),
	}
}

func (s *Set) Add(item string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.m[item] = true
}

func (s *Set) Remove(item string) {
	s.mu.Lock()
	// no error if item is not exist
	delete(s.m, item)
	s.mu.Unlock()
}

func (s *Set) Has(item string) bool {
	s.mu.RLock()
	defer s.mu.RUnlock()
	_, ok := s.m[item]
	return ok
}

func (s *Set) Len() int {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return len(s.m)
}

func (s *Set) Clear() {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.m = map[string]bool{}
}

func (s *Set) IsEmpty() bool {
	if s.Len() == 0 {
		return true
	}
	return false
}

func (s *Set) List() []string {
	s.mu.RLock()
	defer s.mu.RUnlock()
	list := make([]string, 0, s.Len())
	for k := range s.m {
		list = append(list, k)
	}
	return list
}

func (s *Set) SortList(reverse bool) []string {
	list := s.List()
	if reverse {
		sort.Sort(sort.Reverse(sort.StringSlice(list)))
	} else {
		sort.Strings(list)
	}
	return list
}

func main() {
	s := NewSet()
	s.Add("f")
	s.Add("f")
	s.Add("k")
	s.Add("b")
	s.Add("a")
	fmt.Println("length is", s.Len())
	key := "f"
	if s.Has(key) {
		fmt.Println(key, "is exist")
	}
	s.Remove("f")
	fmt.Println(s.List())
	fmt.Println(s.SortList(false))
	s.Clear()
	if s.IsEmpty() {
		fmt.Println("set is empty")
	}
}
```

16.实现一个多生产者和多消费者的消息队列
```go

package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

//产品
type Product struct {
	name  int
	value int
}

//生产者
//stop 标志不为 false，不断地往channel里面放 product，完成之后信号量完成
func producer(wg *sync.WaitGroup, products chan<- Product, name int, stop *bool) {
	for !*stop {
		p := Product{name: name, value: rand.Int()}
		products <- p
		fmt.Printf("producer %v produce a product: %#v\n", name, p)
		time.Sleep(time.Duration(200+rand.Intn(1000)) * time.Millisecond)
	}
	wg.Done()
}

//消费者
//不断地从channel里面取 product，然后作对应的处理，直到通道被关闭，并且 products 里面为空，for循环才会终止
func consumer(wg *sync.WaitGroup, products <-chan Product, name int) {
	for p := range products {
		fmt.Printf("consumer %v consume a product: %#v\n", name, p)
		time.Sleep(time.Duration(200+rand.Intn(1000)) * time.Millisecond)
	}
	wg.Done()
}

func main() {
	var wgp sync.WaitGroup
	var wgc sync.WaitGroup
	stop := false
	m, n := 5, 5                        //生产者和消费者的数量
	products := make(chan Product, 10)  //channel的长度可以是生产速度/消费速度,这里假设生产速度是消费速度的10倍

	//创建生产者
	for i := 0; i < m; i++ {
		go producer(&wgp, products, i, &stop)
		wgp.Add(1)
	}
	//创建消费者
	for i := 0; i < n; i++ {
		go consumer(&wgc, products, i)
		wgc.Add(1)
	}

	time.Sleep(1 * time.Second)
	stop = true     // 设置生产者终止信号
	wgp.Wait()      // 等待生产者退出
	close(products) // 关闭通道
	wgc.Wait()      // 等待消费者退出
}

```
17.循环队列(非并发安全)
```go
package main

import (
	"errors"
	"fmt"
)

type QElemType int

//循环队列存储结构
type SqQueue struct {
	data              []QElemType
	front, rear, size int
}

func NewSqQueue(size int) *SqQueue {
	//当尾指针q.rear的下一个位置是q.front是时，就认为是队满,所以浪费一个空间,所以size+1
	return &SqQueue{
		data: make([]QElemType, size+1),
		size: size + 1,
	}
}

func (q *SqQueue) Len() int {
	return (q.rear - q.front + q.size) % q.size
}

func (q *SqQueue) Put(e QElemType) error {
	if (q.rear+1)%q.size == q.front {
		return errors.New("队列已满！")
	}

	q.data[q.rear] = e
	q.rear = (q.rear + 1) % q.size //循环，不然会越界
	return nil
}

func (q *SqQueue) Get() (e QElemType, err error) {
	if q.IsEmpty() {
		err = errors.New("队列为空，没有数据出队列")
		return
	}
	e = q.data[q.front]
	q.data[q.front] = 0              //清空数据
	q.front = (q.front + 1) % q.size //数据出队列之后，指针移动到即将出队列的元素位置
	return
}

func (q *SqQueue) IsEmpty() bool {
	return q.front == q.rear
}

func main() {
	q := NewSqQueue(3)
	for i := 0; i < 4; i++ {
		if err := q.Put(QElemType((i + 1) * 10)); err != nil {
			fmt.Println(err)
			continue
		}
		fmt.Println("queue length is", q.Len())
	}
	for i := 0; i < 4; i++ {
		if e, err := q.Get(); err != nil {
			fmt.Println(err)
		} else {
			fmt.Println(e)
			fmt.Println("queue length is", q.Len())
		}
	}
	if q.IsEmpty() {
		fmt.Println("queue is empty")
	}
}

```

18.无锁循环队列(并发安全)
```go
package main

import (
	"errors"
	"fmt"
	"runtime"
	"sync/atomic"
	"time"
)

var ErrEmpty = errors.New("queue is empty")
var ErrFull = errors.New("queue is full")
var ErrTimeout = errors.New("queue op. timeout")

type data struct {
	//isFull bool
	stat  uint32 //isFull(读写指针相同时有bug) 改为 stat , 0(可写), 1(写入中), 2(可读)，3(读取中)
	value []byte
}

// BytesQueue .queque is a []byte slice
type BytesQueue struct {
	cap    uint32 //队列容量
	len    uint32 //队列长度
	ptrStd uint32 //ptr基准(cap-1)
	putPtr uint32 // queue[putPtr].stat must < 2
	getPtr uint32 // queue[getPtr].stat may < 2
	queue  []data //队列
}

//NewBtsQueue cap转换为的2的n次幂-1数,建议直接传入2^n数
//如出现 error0: the get pointer excess roll 说明需要加大队列容量(其它的非nil error 说明有bug)
//非 阻塞型的 put get方法 竞争失败时会立即返回
func NewBtsQueue(cap uint32) *BytesQueue {
	bq := new(BytesQueue)
	bq.ptrStd = minCap(cap)
	bq.cap = bq.ptrStd + 1
	bq.queue = make([]data, bq.cap)
	return bq
}

func minCap(u uint32) uint32 { //溢出环形计算需要，得出一个2的n次幂减1数（具体可百度kfifo）
	u-- //兼容0, as min as ,128->127 !255
	u |= u >> 1
	u |= u >> 2
	u |= u >> 4
	u |= u >> 8
	u |= u >> 16
	return u
}

//Len method
func (bq *BytesQueue) Len() uint32 {
	return atomic.LoadUint32(&bq.len)
}

//Empty method
func (bq *BytesQueue) Empty() bool {
	if bq.Len() > 0 {
		return false
	}
	return true
}

//Put method
func (bq *BytesQueue) Put(bs []byte) (bool, error) {
	var putPtr, stat uint32
	var dt *data

	putPtr = atomic.LoadUint32(&bq.putPtr)

	if bq.Len() >= bq.ptrStd {
		return false, ErrFull
	}

	if !atomic.CompareAndSwapUint32(&bq.putPtr, putPtr, putPtr+1) {
		return false, nil
	}
	atomic.AddUint32(&bq.len, 1)
	dt = &bq.queue[putPtr&bq.ptrStd]

	for {
		stat = atomic.LoadUint32(&dt.stat) & 3
		if stat == 0 {
			//可写

			atomic.AddUint32(&dt.stat, 1)
			dt.value = bs
			atomic.AddUint32(&dt.stat, 1)
			return true, nil
		}
		runtime.Gosched()

	}

}

//Get method
func (bq *BytesQueue) Get() ([]byte, bool, error) {
	var getPtr, stat uint32
	var dt *data

	var bs []byte //中间变量，保障数据完整性

	getPtr = atomic.LoadUint32(&bq.getPtr)

	if bq.Len() < 1 {
		return nil, false, ErrEmpty
	}

	if !atomic.CompareAndSwapUint32(&bq.getPtr, getPtr, getPtr+1) {
		return nil, false, nil
	}
	atomic.AddUint32(&bq.len, 4294967295) //^uint32(-1-1)==uint32(0)-uint32(1)
	dt = &bq.queue[getPtr&bq.ptrStd]

	for {
		stat = atomic.LoadUint32(&dt.stat)
		if stat == 2 {
			//可读
			atomic.AddUint32(&dt.stat, 1) // change stat to 读取中
			bs = dt.value
			dt.value = nil
			atomic.StoreUint32(&dt.stat, 0) //重置stat为0
			return bs, true, nil

		}
		runtime.Gosched()

	}

}

// PutWait 阻塞型put,ms 最大等待豪秒数,默认 1000
func (bq *BytesQueue) PutWait(bs []byte, ms ...time.Duration) error {

	var start, end time.Time

	start = time.Now()
	end = start.Add(time.Millisecond * 1000)
	if len(ms) > 0 {
		end = end.Add(time.Millisecond * ms[0])
	}

	for {
		ok, err := bq.Put(bs)
		if ok {
			return nil
		}
		if ErrFull == err {
			time.Sleep(50 * time.Millisecond)
		}

		if time.Now().After(end) {
			return ErrTimeout
		}
	}
}

// GetWait 阻塞型get, ms为 等待毫秒 默认1000
func (bq *BytesQueue) GetWait(ms ...time.Duration) ([]byte, error) {

	var start, end time.Time

	start = time.Now()
	end = start.Add(time.Millisecond * 1000)
	if len(ms) > 0 {
		end = start.Add(time.Millisecond * ms[0])
	}

	for {
		value, ok, err := bq.Get()
		if ok {
			return value, nil
		}

		if ErrEmpty == err {
			time.Sleep(50 * time.Millisecond)
		}

		if time.Now().After(end) {
			return nil, ErrTimeout
		}

	}

}

func main() {
	bq := NewBtsQueue(8)
	bq.Put([]byte("hehe"))
	s, _, _ := bq.Get()
	fmt.Println(string(s))
}
```

30.
```
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}
```
结果:
```
showA
showB
```
分析:
```
首先明确一点 go中没有继承关系。也不应该提及“继承”这个词，其中Trecher并没有继承Propler，而是嵌套People,
而t.ShowA()是一个语法糖，其实t.ShowA() = t.people.ShowA(),也就是说在嵌套结构中，go会优先调用本身方法，
如果本身没有此方法，就回去调用其所包含结构的方法。

本题中，showA()是Teacher不具有的，但是它所嵌套的People具有，因此回调用People.showA(),People.showA()
中调用了*People 的showB()当然会展示“shwoB”，而不是“teacher showB”

引申一点，
如果嵌套有两个结构，并且两个结构具有相同的方法，如何执行的？答案是 编译报错，不支持这种情况的。
```
31.[写入http响应的几种方法的区别](https://stackoverflow.com/questions/37863374/whats-the-difference-between-responsewriter-write-and-io-writestring)


[深入理解反射](https://studygolang.com/articles/12348?fr=sidebar)

32.

大部分编程语言使用固定大小的函数调用栈，常见的大小从64KB到2MB不等。固定大小栈会限制递归的深度，当你用递归处理大量数据时，需要避免栈溢出；除此之外，还会导致安全性问题。与相反,Go语言使用可变栈，栈的大小按需增加(初始时很小)。这使得我们使用递归时不必考虑溢出和安全问题。



33.调度器

Go 调度器是 Go 运行时的一部分，Go 运行时内置在应用程序中。这意味着 Go 调度器在内核之上的用户空间中运行。当 Go 程序启动时，它会为主机上标识的每个虚拟核心提供一个逻辑处理器（P），比如四核八线程，那么就有8个 P，每个 P 都被分配一个系统线程 M，操作系统负责将线程放在一个核心上执行。Goroutines 在 M 上进行上下文切换。Go 调度器中有两个不同的运行队列：全局运行队列(GRQ)和本地运行队列(LRQ)。每个 P 都有一个LRQ，用于管理分配给在P的上下文中执行的 Goroutines，这些 Goroutine 轮流被和 P 绑定的 M 进行上下文切换。GRQ 适用于尚未分配给P的 Goroutines。

在 Go 程序中发生一些事件，比如使用关键字 go、垃圾回收、系统调用、同步和编配，它们允许调度器做出调度决策。

像如果原子、互斥量或通道操作调用将导致 Goroutine 阻塞，调度器可以将之切换到一个新的 Goroutine 去运行。一旦 Goroutine 可以再次运行，它就可以重新排队，并最终在M上切换回来

像网络系统调用这种事件，Goroutine 会被移动到网络轮询器并且处理，M可以从 LRQ 执行另外的 Goroutine，处理完 Goroutine 被移回到 P 的 LRQ 中。

像Goroutine 要执行同步的系统调用，比如基于文件的系统调用或者正在使用 CGO， Goroutine会阻塞M，调度器识别出G1已导致M1阻塞，此时，调度器将M1与P分离，同时也将G1带走。然后调度器引入新的M2来服务P。此时，可以从 LRQ 中选择新的G并在M2上进行上下文切换。

调度器的另一个方面是它是一个任务窃取的调度器。其中一个P的所有 Goroutines 很快就执行完，它会以一定的概率从全局可运行队列偷，或者从其他的P 的本地运行队列偷，这种机制保证 M 不止闲着。

总的来说，调度器的设计让多个 Goroutine 复用同一个系统线程和核心，上下文切换在用户态进行。从操作系统的角度来看，操作系统线程永远不会进入等待状态。因此，在使用系统线程时的开销在使用 Goroutine 时就不存在了。

34.垃圾回收

go1.5开始使用三色标记法，主要流程如下：

- 所有对象最开始都是白色。
- 从 root 开始找到所有可达对象，标记为灰色，放入待处理队列。
- 遍历灰色对象队列，将其引用对象标记为灰色放入待处理队列，自身标记为黑色。
- 处理完灰色对象队列，执行清扫工作。

go1.8开始加入混合写屏障(hybrid write barrier)，垃圾回收和应用程序并发执行。就是在gc跑的过程中，可以监控对象的内存修改，并对对象进行重新标记(实际上也是超短暂的stw，然后对对象进行标记)。

 GC 触发方式有自动检测和用户主动调用。除此之后 Golang 本身还会对运行状态进行监控，如果超过两分钟没有 GC，则触发 GC。监控函数是 sysmon()，在主 goroutine 中启动。


35.go-micro

### 服务端

0. 初始化解析环境变量和命令行参数(可选)
1. 选择通信协议和编码方式(默认http和protobuf)
2. 注册Handler，反射所有信息并保存
3. 启动服务，等待连接到来
4. 注册服务节点信息，定期心跳检查和重新注册服务

### 客户端

0. 初始化解析环境变量和命令行参数(可选)
1. 客户端Call调用方法，封装参数和编码数据
2. 从Selector的选择算法得到服务节点信息(Selector是客户端的的负载均衡，监听注册中心，更新和缓存服务器信息)
3. 根据节点信息从连接池里面拿连接，没有则创建，用完放回池里
4. 发送数据，服务端接收和解码数据，找到相应的实例和方法，利用反射调用具体方法->编码返回数据->发送给客户端
5. 解码返回数据