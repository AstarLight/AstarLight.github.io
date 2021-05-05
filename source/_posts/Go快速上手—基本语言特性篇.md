---
title: Go快速上手—基本语言特性篇
tags: [go]	
categories: 技术
---

## 编译和运行
- go run main.go  直接运行
- go build -o test main.go  指定文件名输出二进制文件

## 基础语法


```go
package main
import (
	"fmt"
	"reflect"
	"os"
	"errors"
)

func add(num int){
	num += 1
}

func realAdd(num *int){
	*num += 1
}

func add2 (n1 int, n2 int) (int, int) {
	return n1+n2, n1-n2
}

func say(name string) error {
	if len(name) == 0 {
		return errors.New("error:name is null")
	}
	fmt.Println("hello ", name)
	return nil
}

type Student struct {
	name string
	age int
	info map[string]string
}

func (stu *Student) hello(person string) string {
	return fmt.Sprintf("hello %s, i am %s", person, stu.name)
}

func get(index int) (ret int) {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("some err happens!", r)
		}
		ret = -1
	}()
	arr := [3]int{2,3,4}
	return arr[index]
}

func main()  {
	var name string = "Hello world!"
	var a = 1 //默认是整型
	b := 2
	c := "my name"
	var d int // 默认为0
	var e byte = 'a'
	var f int8 = 1
	var g float32 = 1.0
	ok := false

	str1 := "Go学习"  // 中文一个字一般占3个字节
	fmt.Println(name)
	fmt.Println(a,b,c,d,e,f,g,ok) // 1 2 my name 0 97 1 1 false
	fmt.Println(reflect.TypeOf(str1[2]).Kind()) // 打印类型,uint8
	fmt.Printf("%d %c\n", str1[1], str1[1])
	fmt.Println("len(str1)=", len(str1)) // 8字节

	//将str1切割成rune数组，一个元素是一个汉字或字母
	//转换成 []rune 类型后，字符串中的每个字符，无论占多少个字节都用 int32 来表示，因而可以正确处理中文。
	//rune 类型，代表一个 UTF-8 字符，当需要处理中文、日文或者其他复合字符时，则需要用到 rune 类型。rune 类型等价于 int32 类型。
	runeArr := []rune(str1)  // 等价于 runeArr := []int32(str1)
	fmt.Println(reflect.TypeOf(runeArr[2]).Kind()) // int32
	fmt.Printf("%d %c\n", runeArr[2], runeArr[2])  // 23398 学
	fmt.Println("len(runeArr)=", len(runeArr)) // 4，元素个数

	// 声明数组
	var arr [5]int 
	var arr2 [5][5]int // 二维
	// 声明+初始化
	var arr3 = [5]int{1,2,3,4,5}
	arr4 := [5]int{1,2,3,4,5}
	for i := 0; i < len(arr3); i++ {
		arr3[i] += 100
	}
	fmt.Println(arr)
	fmt.Println(arr2)
	fmt.Println(arr3)
	fmt.Println(arr4)

	//数组的长度不能改变，如果想拼接2个数组，或是获取子数组，需要使用切片。
	// 切片是数组的抽象。 切片使用数组作为底层结构。
	slice1 := make([]float32, 0)  // 长度为0的切片
	slice2 := make([]float32, 3, 5) // 长度为3容量为5的切片
	fmt.Println(len(slice1), cap(slice2))  // 0 5


	slice2 = append(slice2, 1, 2, 3, 4) 
	fmt.Println(len(slice2), cap(slice2)) // 7 12
	fmt.Println(slice2)
	sub1 := slice2[3:]  //[1 2 3 4]
	sub2 := slice2[:4] // [0 0 0 1]
	sub3 := slice2[1:5] // [0 0 1 2]
	combine := append(sub1, sub2...) ///sub2... 是切片解构的写法，将切片解构为 N 个独立的元素。[1 2 3 4 0 0 0 1]

	fmt.Println(sub1)
	fmt.Println(sub2)
	fmt.Println(sub3)
	fmt.Println(combine)

	m1 := make(map[string]int)
	// 声明加初始化
	m2 := map[string]int {
		"grade" : 100,
		"level" : 1,
	}
	m1["grade"] = 101
	fmt.Println(m1)
	fmt.Println(m2)

	ss := "golang"
	var p *string = &ss // p 是指向ss的指针
	*p = "golang-gogo"
	fmt.Println(ss, *p)  // golang-gogo golang-gogo

	//一般来说，指针通常在函数传递参数，或者给某个类型定义新的方法时使用。
	//Go 语言中，参数是按值传递的，如果不使用指针，函数内部将会拷贝一份参数的副本，对参数的修改并不会影响到外部变量的值。
	//如果参数使用指针，对参数的传递将会影响到外部变量。
	a1 := 1
	add(a1)
	fmt.Println(a1) // 1
	realAdd(&a1)
	fmt.Println(a1)  // 2

	// Go 语言中没有枚举(enum)的概念，一般可以用常量的方式来模拟枚举。
	type Gender int8   // 跟c的typedef一样的
	const (
		MALE Gender = 1
		FEMALE Gender = 2
	)
	//Go 语言的 switch 不需要 break，匹配到某个 case，执行完该 case 定义的行为后，默认不会继续往下执行。如果需要继续往下执行，需要使用 fallthrough
	gender := MALE
	switch gender {
	case MALE:
		fmt.Println("i am male")
		fallthrough
	case FEMALE:
		fmt.Println("i am female")
	default:
		fmt.Println("unknown")
	}
	//output:
	//i am male
	//i am female

	// 遍历
	nums := []int{10, 20, 30}
	for i, num := range nums {
		fmt.Println(i, num)
	}

	mm2 := map[string]string {
		"key1": "val1",
		"key2": "val2",
	}

	for key,val := range mm2 {
		fmt.Println(key, val)
	}

	for i, num := range combine {
		fmt.Println(i, num)
	}

	// 如果调用成功，error 的值是 nil，
	// 如果调用失败，例如文件不存在，我们可以通过 error 知道具体的错误信息。
	_, err := os.Open("a.txt")
	if err != nil{
		fmt.Println(err)
	}

	err = say("")
	if err != nil{
		fmt.Println(err)
	}


	// error 往往是能预知的错误，但是也可能出现一些不可预知的错误，
	// 例如数组越界，这种错误可能会导致程序非正常退出，在 Go 语言中称之为 panic。
	fmt.Println(get(5))
	fmt.Println("finished")

	// 在 get 函数中，使用 defer 定义了异常处理的函数，在协程退出前，会执行完 defer 挂载的任务。因此如果触发了 panic，控制权就交给了 defer。
	// 在 defer 的处理逻辑中，使用 recover，使程序恢复正常，并且将返回值设置为 -1，在这里也可以不处理返回值，如果不处理返回值，返回值将被置为默认值 0。

	//结构体和方法
	stu := &Student { 
		name: "Tom",
		age: 19,
	}
	msg := stu.hello("Ken")
	fmt.Println(msg)

	// 可以new方法来new 实例
	stu2 := new(Student)
	stu2.name = "kk"
	fmt.Println(stu2.hello("Alice"))

	// 空接口,定义了一个没有任何方法的空接口，那么这个接口可以表示任意类型
	mmp := make(map[string]interface{})
	mmp["name"] = "james"
	mmp["age"] = 20
	mmp["score"] = [3]int{1,2,3}
	mmp["info"] = m2
	fmt.Println(mmp)  // map[age:20 info:map[grade:100 level:1] name:james score:[1 2 3]]

}

```

## 方法和接口
在 Go 中，只需使用大写标识符，即可公开方法，使用非大写的标识符将方法设为私有方法。


```go
package main

// main.go

import (
	"fmt"
)

type triangle struct {
	size int
}

func (t triangle) perimeter() int {
	return t.size * 3
}

// 指针可以直接修改变量值，而不发生复制
func (t *triangle) doubleSize() {
	t.size *= 2
}

type colorTriangle struct {
	triangle
	color string
}

// 重载方法，没有重载该方法时，调用的是triangle的perimeter()
func (t colorTriangle) perimeter() int {
	return t.size * 2 * 3
}


func main() {
	t := triangle{3}
	fmt.Println("perimeter:", t.perimeter())
	t.doubleSize()
	fmt.Println("doublesize:", t.perimeter());

	s := colorTriangle{triangle{4}, "blue"}
	fmt.Println("perimeter color:", s.perimeter())
	fmt.Println("perimeter normal:", s.triangle.perimeter())

}

```

输出
```
perimeter: 9
doublesize: 18
perimeter color: 24
perimeter normal: 12
```

Go 中的接口是一种用于表示其他类型的行为的数据类型。 在你使用接口时，你的基本代码将变得更加灵活、适应性更强，因为你编写的代码未绑定到特定的实现。接口就是只声明了函数，但是函数尚未具体实现。

Go 中的接口是一种抽象类型，只包括具体类型必须拥有或实现的方法。 

```
type Shape interface {
    Perimeter() float64
    Area() float64
}
```

### 接口interface

一般而言，接口定义了一组方法的集合，接口不能被实例化，一个类型可以实现多个接口。


这里的接口跟```C++```的多态性质比较像，一个interface可以指向任何实现了其定义的方法的struct，这就好比C++的基类指针指向子类对象，实现多态。

==对于任何数据类型，只要它的方法集合中完全包含了一个接口的全部特征（即全部的方法），那么它就一定是这个接口的实现类型。==

```go
package main

import (
    "fmt"
)

type Person interface {
	getName() string
}

type Student struct {
	name string 
	age int
}

func (stu *Student) getName() string {
	return stu.name
}

type Worker struct {
	name string 
	gender string
}

func (wk * Worker) getName() string {
	return wk.name
}


func main() {
	var stu = Student{
		name: "Tom",
		age: 18,
	}
	var p Person = &stu  // 接口转为实例
	fmt.Println(p.getName())

	var wk = Worker{
		name: "Ken",
		gender: "Male",
	}

	p = &wk  //接口转为实例
	fmt.Println(p.getName())

	//如果定义了一个没有任何方法的空接口，那么这个接口可以表示任意类型
	m := make(map[string]interface{})
	m["name"] = "Alice"
	m["age"] = 19
	m["score"] = [3]int{98, 99, 85}
	fmt.Println(m)
}
```

输出
```
Tom
Ken
map[age:19 name:Alice score:[98 99 85]]
```

怎样判定一个数据类型的某一个方法实现的就是某个接口类型中的某个方法呢？

这有两个充分必要条件，一个是“两个方法的签名需要完全一致”，另一个是“两个方法的名称要一模一样”。

两个方法的签名需要完全一致，这就是表示方法的实现的形参、顺序、返回值必须一致，才会有相同的方法签名。

### 扩展现有的方法

现有的实现，io.Copy，将http get回来的数据打印到终端上。现在我们希望重载这个Copy方法，以我们需要的方式打印返回结果。
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
)

func main() {
    resp, err := http.Get("https://api.github.com/users/microsoft/repos?page=15&per_page=5")
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }

    io.Copy(os.Stdout, resp.Body)
}
```

输出
```
[{"id":276496384,"node_id":"MDEwOlJlcG9zaXRvcnkyNzY0OTYzODQ=","name":"-Users-deepakdahiya-Desktop-juhibubash-test21zzzzzzzzzzz","full_name":"microsoft/-Users-deepakdahiya-Desktop-juhibubash-test21zzzzzzzzzzz","private":false,"owner":{"login":"microsoft","id":6154722,"node_id":"MDEyOk9yZ2FuaXphdGlvbjYxNTQ3MjI=","avatar_url":"https://avatars2.githubusercontent.com/u/6154722?v=4","gravatar_id":"","url":"https://api.github.com/users/microsoft","html_url":"https://github.com/micro
....
```

io.Copy的实现
```go
func Copy(dst Writer, src Reader) (written int64, err error)

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

注意到Writer是空接口，里面实现了write方法。因此我们重新定义一个Writer接口，幷实现这个Write方法
```go
package main

import (
    "fmt"
    "io"
    "net/http"
    "os"
	"encoding/json"
)


type GitHubResponse []struct {
	FullName string `json:"full_name"`
}

type customWriter struct {}

func (w customWriter) Write(p []byte) (n int, err error) {
	var resp GitHubResponse
	json.Unmarshal(p, &resp)
	for _, r := range resp {
		fmt.Println(r.FullName)
	}
	return len(p), nil
}

func main() {
    resp, err := http.Get("https://api.github.com/users/microsoft/repos?page=15&per_page=5")
    if err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
	writer := customWriter{}
    io.Copy(writer, resp.Body)
}
```
输出
```go
microsoft/aed-content-nasa-su20
microsoft/aed-external-learn-template
microsoft/aed-go-learn-content
microsoft/aed-learn-template
```



## defer和panic

defer 语句会推迟函数（包括任何参数）的运行，直到包含 defer 语句的函数完成。 通常情况下，当你想要避免忘记任务（例如关闭文件或运行清理进程）时，可以推迟某个函数的运行。

内置 panic() 函数会停止正常的控制流。 所有推迟的函数调用都会正常运行。 进程会在堆栈中继续，直到所有函数都返回。 然后，程序会崩溃并记录日志消息。 此消息包含错误和堆栈跟踪，有助于诊断问题的根本原因。

这是panic和defer组合使用的例子

```go
package main

// main.go

import (
	"fmt"
)


func main() {
	gg(0)
	fmt.Println("Program finished succ!")
}

func gg(i int) {
	if (i > 3) {
		fmt.Println("throw panic!!!")
		panic("Panic in gg")
	}
	defer func() {
		fmt.Println("Defer in gg, i=", i)
	}()

	fmt.Println("Printing in gg. i=", i)
	gg(i+1)
}
```

输出
```
rinting in gg. i= 0
Printing in gg. i= 1
Printing in gg. i= 2
Printing in gg. i= 3
throw panic!!!
Defer in gg, i= 3
Defer in gg, i= 2
Defer in gg, i= 1
Defer in gg, i= 0
panic: Panic in gg

goroutine 1 [running]:
main.gg(0x4)
        /Users/junshili/Desktop/golearn/main.go:18 +0x195
main.gg(0x3)
        /Users/junshili/Desktop/golearn/main.go:25 +0xfe
main.gg(0x2)
        /Users/junshili/Desktop/golearn/main.go:25 +0xfe
main.gg(0x1)
        /Users/junshili/Desktop/golearn/main.go:25 +0xfe
main.gg(0x0)
        /Users/junshili/Desktop/golearn/main.go:25 +0xfe
main.main()
        /Users/junshili/Desktop/golearn/main.go:11 +0x2e
```

注意输出“Defer in gg”是采用逆序（后进先出）。最后的```fmt.Println("Program finished succ!")```并没有执行，这是因为panic出现后，程序就会退出。

## recover

Go 提供内置函数 recover()，允许你在出现紧急状况之后重新获得控制权。 只能在已推迟的函数中使用此函数。 如果调用 recover() 函数，则在正常运行的情况下，它会返回 nil，没有任何其他作用。
```go
package main

// main.go

import (
	"fmt"
)


func main() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered in main", r)
		}
	}()
	gg(0)
	fmt.Println("Program finished succ!")
}

func gg(i int) {
	if (i > 3) {
		fmt.Println("throw panic!!!")
		panic("Panic in gg")
	}
	defer func() {
		fmt.Println("Defer in gg, i=", i)
	}()

	fmt.Println("Printing in gg. i=", i)
	gg(i+1)
}
```

输出
```
Printing in gg. i= 0
Printing in gg. i= 1
Printing in gg. i= 2
Printing in gg. i= 3
throw panic!!!
Defer in gg, i= 3
Defer in gg, i= 2
Defer in gg, i= 1
Defer in gg, i= 0
Recovered in main Panic in gg
```

这个版本跟上一个panic+defer的版本相比，输出少了堆栈信息，控制流同样是中断了，输出顺序是一样的。

==panic 和 recover 的组合是 Go 处理异常的惯用方式。 其他编程语言使用 try/catch 块。 Go 首选此处所述的方法。==



## 并发编程goroutine

编写并发程序时最大的问题是在进程之间共享数据。Go 是通过 channel 来回传递数据的。 这意味着只有一个活动 (goroutine) 有权访问数据，设计上不存在争用条件。

单线程写法
```go
package main

// gorountine 并发编程

import (
	"fmt"
	"time"
)


func download(url string) {
	fmt.Println("start donwload,url=", url)
	time.Sleep(time.Second * 5)  // 模拟耗时
}

func main() {
	fmt.Println("start all download, time=", time.Now().Format("2006-01-02 15:04:05"))
	t1 := time.Now()
	for i := 0; i < 10; i++ {
		download("www.test_go.com/index" + string(i + '0'))
	}
	t2 := time.Since(t1) / time.Second
	fmt.Printf("download done, time_use=%ds\n", t2)
}
```

输出

```
start all download, time= 2021-04-29 23:45:46
start donwload,url= www.test_go.com/index0
start donwload,url= www.test_go.com/index1
start donwload,url= www.test_go.com/index2
start donwload,url= www.test_go.com/index3
start donwload,url= www.test_go.com/index4
start donwload,url= www.test_go.com/index5
start donwload,url= www.test_go.com/index6
start donwload,url= www.test_go.com/index7
start donwload,url= www.test_go.com/index8
start donwload,url= www.test_go.
```

gorountine写法

```go
package main

// goroutine 并发编程

import (
	"fmt"
	"sync"
	"time"
)

var wg sync.WaitGroup


func download(url string) {
	fmt.Println("start donwload,url=", url)
	time.Sleep(time.Second * 5)  // 模拟耗时
	wg.Done()  
}

func main() {
	fmt.Println("start all download, time=", time.Now().Format("2006-01-02 15:04:05"))
	t1 := time.Now()
	for i := 0; i < 10; i++ {
		wg.Add(1)  // 为 wg 添加一个计数，wg.Done()，减去一个计数。
		go download("www.test_go.com/index" + string(i + '0'))  //启动新的协程并发执行 download 函数。
	}
	wg.Wait()  //等待所有的协程执行结束。
	t2 := time.Since(t1) / time.Second

	fmt.Printf("download done, time_use=%ds\n", t2)
}
```

输出
```
start all download, time= 2021-04-29 23:49:27
start donwload,url= www.test_go.com/index9
start donwload,url= www.test_go.com/index0
start donwload,url= www.test_go.com/index1
start donwload,url= www.test_go.com/index2
start donwload,url= www.test_go.com/index6
start donwload,url= www.test_go.com/index7
start donwload,url= www.test_go.com/index8
start donwload,url= www.test_go.com/index3
start donwload,url= www.test_go.com/index4
start donwload,url= www.test_go.com/index5
download done, time_use=5s
```

许多程序喜欢使用匿名函数来创建 goroutine，如下所示：
```
func main(){
    login()
    go func() {
        launch()
    }()
}
```

## 利用channel进行协程间通信

Channel是Go中的一个核心类型，你可以把它看成一个管道，通过它并发核心单元就可以发送或者接收数据进行通讯(communication)。

Channel可以作为一个先入先出(FIFO)的队列，接收的数据和发送的数据的顺序是一致的。

channel使用的语法

```
ch <- x // sends (or write) x through channel ch
x = <-ch // x receives (or reads) data sent to the channel ch
<-ch // receives data, but the result is discarded
```

往一个已经被close的channel中继续发送数据会导致run-time panic。

往nil channel中发送、接收数据会一致被阻塞着。

从一个被close的channel中接收数据不会被阻塞，而是立即返回，接收完已发送的数据后会返回元素类型的零值(zero value)。

如前所述，你可以使用一个额外的返回参数来检查channel是否关闭。如果OK 是false，表明接收的x是产生的零值，这个channel被关闭了或者为空。
```
x, ok := <-ch
```

循环处理channel的消息,如果通过range读取，channel关闭后for循环会跳出：
```
for i := range c {
    fmt.Println(i)
}
```


```go
package main

// gorountine 并发编程，利用channel进行协程间通信

import (
	"fmt"
	"time"
)

var ch = make(chan string, 10) // 创建大小为10 的缓冲信道

//如果没有设置容量，或者容量设置为0, 说明Channel没有缓存，只有sender和receiver都准备好了后它们的通讯(communication)才会发生(Blocking)。如果设置了缓存，就有可能不发生阻塞， 
//只有buffer满了后 send才会阻塞， 而只有缓存空了后receive才会阻塞。一个nil channel不会通信。


func download(url string) {
	fmt.Println("start donwload,url=", url)
	time.Sleep(time.Second * 5)  // 模拟耗时
	msg := url + " call"
	ch <- msg  // 发送值msg到Channel ch中
}

func main() {
	fmt.Println("start all download, time=", time.Now().Format("2006-01-02 15:04:05"))
	t1 := time.Now()
	for i := 0; i < 10; i++ {
		go download("www.test_go.com/index" + string(i + '0'))
	}
	for i := 0; i < 10; i++ {
		msg := <-ch  //// 从Channel ch中接收数据，并将数据赋值给msg
		fmt.Println("finish", msg)
	}
	t2 := time.Since(t1) / time.Second

	fmt.Printf("download done, time_use=%ds\n", t2)
	close(ch)
}
```


Go 中 channel 的一个有趣特性是，在使用 channel 作为函数的参数时，可以指定 channel 是要发送数据还是接收数据。 随着程序的增长，可能会使用大量的函数，这时候，最好记录每个 channel 的意图，以便正确使用它们。

```
chan<- int // it's a channel to only send data
<-chan int // it's a channel to only receive data
```

两个函数的示例，一个函数用于读取数据，另一个函数用于发送数据：

```go
package main

import (
    "fmt"
)

func send(ch chan<-string, message string) {
	fmt.Printf("sending: %#v\n", message)
	ch <- message
}

func read(ch <-chan string) {
	fmt.Printf("recieving: %#v\n", <-ch)
}

func main() {
	ch := make(chan string, 1)
	send(ch, "hello world")
	read(ch)
}
```

```
func read(ch <-chan string) {
    fmt.Printf("Receiving: %#v\n", <-ch
    ch <- "Bye!"
}
```
如果试图使用一个 channel 在一个仅用于接收数据的 channel 中发送数据，将会出现编译错误。 
```
# command-line-arguments
./main.go:12:5: invalid operation: ch <- "Bye!" (send to receive-only type <-chan string)
```

报错
```
# command-line-arguments
./main.go:12:5: invalid operation: ch <- "Bye!" (send to receive-only type <-chan string)
```

## 多路复用

有时候我们需要处理多个channel，此时可以使用select来做多路复用，需要等待事件发生。

select 语句的工作方式类似于 switch 语句，但它适用于 channel。 它会阻止程序的执行（阻塞），直到它收到要处理的事件。 如果它收到多个事件，则会随机选择一个。

select 语句的一个重要方面是，它在处理事件后完成执行。 如果要等待更多事件发生，则可能需要使用循环。

```go
package main

import (
    "fmt"
	"time"
)

func process(ch chan string) {
	time.Sleep(3 * time.Second)
	ch <- "Done processing!"
}

func replicate(ch chan string) {
	time.Sleep(time.Second)
	ch <- "Done replicating!"
}

func main() {
	ch1 := make(chan string)
	ch2 := make(chan string)
	go process(ch1)
	go replicate(ch2)

	for i := 0; i < 2; i++ {
		select {
		case process := <-ch1:
			fmt.Println(process)
		case replicate := <-ch2:
			fmt.Println(replicate)
		}
	}
}
```

输出
```
Done replicating!
Done processing!
```

## Timer和Ticker

Ticker比较有用，可以周期性执行某些逻辑

```go
package main

// gorountine 并发编程，Timer和Ticker

import (
	"fmt"
	"time"
)


func main() {
	
	timer1 := time.NewTimer(time.Second * 2)  // timer是一个定时器，代表未来的一个单一事件，你可以告诉timer你要等待多长时间
	go func() {
		<-timer1.C
		fmt.Println("Timer expire")
	}()
	
	stop1 := timer1.Stop()
	
	if stop1 {
		fmt.Println("Timer stop")
	}

	//ticker是一个定时触发的计时器，它会以一个间隔(interval)往Channel发送一个事件(当前时间)，
	//而Channel的接收者可以以固定的时间间隔从Channel中读取事件。

	ticker := time.NewTicker(time.Second * 2)
	go func() {
		for t := range ticker.C {
			fmt.Println("Tick,", t.Format("2006-01-02 15:04:05"))
			//似timer, ticker也可以通过Stop方法来停止。一旦它停止，接收者不再会从channel中接收数据了。
		}
	}()

	for {
		time.Sleep(time.Second)
	}
	
}
```

输出
```
Timer stop
Tick, 2021-04-30 00:43:59
Tick, 2021-04-30 00:44:01
Tick, 2021-04-30 00:44:03
Tick, 2021-04-30 00:44:05
Tick, 2021-04-30 00:44:07
Tick, 2021-04-30 00:44:09
Tick, 2021-04-30 00:44:11
Tick, 2021-04-30 00:44:13
```

## Package和Modules
### Package
一般来说，一个文件夹可以作为 package，同一个 package 内部变量、类型、方法等定义可以相互看到。

如我们新建一个文件 calc.go， main.go 平级

```go
package main

// main.go

import (
	"fmt"
)


func main() {
	
	fmt.Println(add(1,2));
}
```

```go
package main

// calc.go

func add(n1 int, n2 int) int {
	return n1 + n2
}
```


go build -o test main.go calc.go  //文件名顺序并不敏感，或者go run .

### Modules

在项目文件夹下执行go mod init example

初始化一个 Module，名字为example

```go
package main

// main.go

import (
	"fmt"
	"rsc.io/quote"
)


func main() {
	mt.Println(quote.Hello())
}
```

编译或者go run都会触发触发第三方包 rsc.io/quote的下载，具体的版本信息也记录在了go.mod中


考虑这样的项目
```
../golearn/
├── go.mod
├── main.go
└── mypkg
    └── mypkg.go

````

我的main.go需要调用mypkg/mypkg.go的函数，可以这样操作，先在golearn路径下生成一个module：```go mod init example```，然后通过module/目录名的方式调用，注意mypkg.go里定义的函数首字母需要大写，这样才表示该函数是public的，可以供外界调用。

**Go 语言也有 Public 和 Private 的概念，粒度是包。如果类型/接口/方法/函数/字段的首字母大写，则是 Public 的，对其他 package 可见，如果首字母小写，则是 Private 的，对其他 package 不可见。**

main.go
```go
package main


import (
        "fmt"
        "example/mypkg"
)

func main() {
        fmt.Println("test, ", mypkg.Add(1,2))

}

```

## 日志模块

```go
package main


import (
        "log"
        "os"
)

func main() {
        file, err := os.OpenFile("info.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
        if err != nil {
                log.Fatal(err)  // fatal会直接退出本进程
        }

        defer file.Close()

        log.SetOutput(file)
        log.Print("hey, logging")


}

```

日志记录格式
```
2021/04/30 14:59:45 hey, logging
```