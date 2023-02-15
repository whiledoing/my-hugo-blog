---
title: "Learn Golang"
date: 2022-11-30T11:18:59+08:00
author: "whiledoing"
tags: ["golang", "programming-language"]
categories: ["programming"]
---

学习 go-lang 的笔记，主要的学习站点：

- [go-lang-articles](https://golang.org/doc/#articles)
- [go-lang-blog](https://blog.golang.org/)
- [go-vs-python](http://govspy.peterbe.com/)
- [go-by-example](https://gobyexample.com/)

<!--more-->

## go-by-example

学习一下[go by example](https://gobyexample.com/switch)

### switch

switch 可以当做 if/else 来使用, 注意 case 的后面需要加入冒号：

```go
switch {
case t.Hour() < 12:
    fmt.Println("It's before noon")
case t.Hour() < 18:
    fmt.Println("It's noon")
default:
    fmt.Println("It's evening")
}
```

另外一个有用的用法，利用 switch 作为 type 的选择器，在 interface 的选择中很有用：

```go
whatAmI := function(i interface{}) {
    switch t := i.(type) {
    case bool:
        fmt.Println("I'm a bool")
    case int:
        fmt.Println("I'm a int")
    default:
        // 使用%T表示变量为类型参数
        fmt.Println("Don't know type %T\n", t)
    }
}
whatAmI(true)
whatAmI(1)
whatAmI("hello")
```

### slice

数组类型，需要明确指定数组的纬度：

```go
// 数组的纬度记录在类型之前，和一般的语言不太一样
// 数组的类型包括当前数组的长度，注意[5]int和[10]int是不一样的
var a [5]int = [5]int{1,2,3,4}
// 自动计算长度的数组
b := [...]int{1,2,3,4}
var c [10]int
```

如果不指定纬度，就变成了 slice 类型，就是列表，更多可以参考[slice-intro](https://blog.golang.org/slices-intro)

slice 内存结构图：![memory-of-slice](https://blog.golang.org/slices-intro/slice-1.png)

```go
s := []byte{'a', 'b', 'c', 'd'}

// 使用make可以指定len和cap
s2 := make([]int, 10, 20)
// slice的切片公用数据：slice结构类似 {point_to_array, len, cap}
s3 := s2[:2]

// append的代码大致如下
function AppendByte(slice []byte, data ...byte) []byte {
    m := len(slice)
    n := m + len(data)

    if n > cap(slice) {
        // n+1是为了防止长度为0
        newSlice := make([]byte, (n+1)*2)
        copy(newSlice, slice)
        slice = newSlice
    }

    // slice数据还是原来的内容，这样子让slice的长度还是n，而不是扩展后的长度
    slice = slice[:n]

    // 填充新的数值
    copy(slice[m:n], data)
    return slice
}

// 所以使用append操作时候，需要记录返回的结果
s4 := append(s2, 100, 200)
// 这个语法等于是list的extend操作，注意点好放在后面，类似于python的列表解析
s5 := append(s2, s4...)
```

### range

go 的 range 操作支持多模态，如果一个参数就是 value 或者是 map 的 key-list，如果是两个，就是 enumerate 的概念，或者是 map 的 items 的概念：

```go
nums := []{1,2,3,4}

// ignore index
for _, value := range nums {
    if value == 3 {
        fmt.Println(value)
    }
}

// just index in slice
for index := range nums {
    fmt.Println(index)
}

kvs := map[string]string {"a": "apple", "b": "banana"}
for k, v := range kvs {
    fmt.Println(k, v)
}

// only keys
for k := range kvs {
    fmt.Println(k)
}
```

### error

go 的 `error` 机制，类似于 C 的方式，在返回值中透传。个人觉得还是挺好的，可以保持调用函数后，优先判断错误，然后短路返回的调用风格。代码逻辑更健壮,但一定程度上也增加了冗余.

go 的 `error` 其实是一个 `interface` 只要定义对应的`Error() string`方法签名,就是一个自定义的 `error` 类：

```go
func (e *argError) Error() string {
    return fmt.Sprintf("%d - %s", e.arg, e.prob)
}

func testError() {
    testFunc := func (i int) (int, error) {
        switch i {
        case 10:
            // 构造一个自定义error的pointer
            return 0, &argError{i, "can't work"}
        case 11:
            // errors.New返回的是一个errors.errorString的指针
            return 0, errors.New("can't work two")
        case 12:
            // 加入了printf的errors.New调用.
            return 0, fmt.Errorf("can't work three of value: %v", i)
        default:
            return i+3, nil
        }
    }

    if _, err := testFunc(10); err != nil {
        fmt.Println("failed, ", err)
    }

    if _, err := testFunc(11); err != nil {
        fmt.Println("failed, ", err)
    }

    if _, err := testFunc(12); err != nil {
        fmt.Println("failed, ", err)
    }
}
```

为什么 `error` 默认都使用指针，根据这个[custom-errors-in-golang-and-pointer-receivers](https://stackoverflow.com/questions/50333428/custom-errors-in-golang-and-pointer-receivers)的回答。一个可能的考虑点是，使用 `error` 进行逻辑判定时, 会进行等号检测，需要判定的是 is 概念，而不是值相同的概念。所以，使用指针表示错误，可以直接用等号进行 is 检测。

### channel

channel 真是 go-lang 的精髓。

- none-buffer 的通道，必须要有数据等待接受，才可以写入；而 buffer 的通道，可以直接写入，不需要有数据接收。
- 使用 select 可以**同步等待多个通道**或者使用**default**语句，表示等待不到数据的默认选择；或者使用`time.After(time.Seconds)`表示等待一定时间的超时。
- `close`一个通道，表示通道内不会再有更多的数据写入，接受通道数据的协程，可以感知这种情况。如果使用`for job := range jobs`的方式等待通道，会在通道关闭后结束，非常赞的语法糖。使用这个技巧，可以非常容易的实现**协程池**，多个协程并发的等待队列的工作任务，实现参考[worker-pools](https://gobyexample.com/worker-pools)

```go
func ping(pings chan<- string, msg string) {
    pings <- msg
}

// 这里pings只负责输出；而pongs负责输入
func pong(pings <-chan string, pongs chan<- string) {
    msg := <-pings
    pongs <- msg
}

func main() {
    pings := make(chan string, 1)
    pongs := make(chan string, 1)

    ping(pings, "hello world")
    pong(pings, pongs)

    go func() {
        time.Sleep(time.Second)
        c1 <- "v1"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        c2 <- "v2"
    }()

    // select非常强大，多路选择
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <- c1:
            fmt.Println("c1: ", msg1)
        case msg2 := <- c2:
            fmt.Println("c2: ", msg2)
        case <-time.After(time.Second):
            fmt.Println("timeout")
        }
    }

    jobs := make(chan int, 5)
    done := make(chan bool)

    go func() {
        // if job, more := <-jobs; !more {
        //     fmt.Println("all jobs done!")
        //     done <- true
        //     return
        // } else {
        //     fmt.Println("received job: ", job)
        // }

        // 和上面的代码一致，使用range简化了逻辑
        for job := range jobs {
            fmt.Println("received job: ", job)
        }
        done <- true
    }()

    for i := 0; i < 3; i++ {
        jobs <- i
    }

    // close关闭了通道，使用通道接受数据的协程会感知
    close(jobs)
    <- done
    fmt.Println("jobs done")
}
```

使用`WaitGroup`做为同步机制，控制多个协程的执行计数：

```go
// 注意使用WaitGroup需要传递指针，因为会修改其数据
worker := func(id int, wg *sync.WaitGroup) {
    // defer非常精髓，保证一定可以退出执行
    defer wg.Done()

    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second)
    fmt.Printf("Worker %d done\n", id)
}

wg := sync.WaitGroup{}

for i := 0; i < 4; i++ {
    wg.Add(1)
    go worker(i, &wg)
}

// 到了这里，一定执行了4次Add，必须要执行4次的Done操作才可以结束阻塞等待
wg.Wait()
```

使用 go 的 channel 阻塞机制，可以很容易实现令牌桶限流。每隔一段时间就往 channel 中放入令牌，而处理请求时候，必须有令牌才可以放行。基于 channel 的 buffer 机制，可以控制初始的令牌个数：

```go
// 初始个数
burstyLimiter := make(chan time.Time, 3)
for i := 0; i < 3; i++ {
    burstyLimiter <- time.Now()
}

// 每隔一段时间投放数据到令牌桶中
go func() {
    for t := range time.Tick(200 * time.Millisecond) {
        burstyLimiter <- t
    }
}()

// 投放任务，使用close便于使用range
jobs := make(chan int, 5)
for i := 0; i < 5; i++ {
    jobs <- i
}
close(jobs)

// 等待令牌桶中有数据才可以通过阻塞
for job := range jobs {
    t := <- burstyLimiter
    fmt.Printf("get job %v: %v\n", job, t)
}
```

go 的同步哲学：通过 channel 共享内存，将数据只放在一个协程中进行处理，通过 channel 进行同步。

考虑并发读写请求一个 map，正常的实现是在读写时，給 map 加锁。而 go 的方式是将 map 放在一个状态管理协程中，读写操作都变成任务放入请求队列中，在状态协程处理完毕后，将数据通过通道返回給请求协程。参考[stateful-goroutines](https://gobyexample.com/stateful-goroutines)

```go
// 读请求的任务，需要透传交互用的channel
type readOp struct {
    key int
    resp chan int
}

// 写请求的任务，同样需要channal进行交互
type writeOp struct {
    key int
    value int
    resp chan bool
}

// 读写的请求队列，不是buffer的队列很精髓：如果处理任务的协程没有在等待任务，是不可以写入的
reads := make(chan readOp)
writes := make(chan writeOp)

// go的同步哲学：数据只放在一个协程中管理。通过channel进行通信
go func() {
    m := make(map[int]int)
    for {
        // 可实现全局mutex功能，只有一个任务可进入读或者写状态
        // 如果接受到一个读写请求，读写的等待都消除进行case的处理。这时，新的读写请求进不来，因为没有等待就没有写入。
        select {
        case read := <-reads:
            read.resp <- m[read.key]
        case write := <-writes:
            m[write.key] = m[write.value]
            write.resp <- true
        }
    }
}()

var readCnt, writeCnt uint64
for i := 0; i < 100; i++ {
    go func() {
        readResp := make(chan int)
        for {
            reads <- readOp{ key: rand.Intn(5), resp: readResp }
            <- readResp
            atomic.AddUint64(&readCnt, 1)
            time.Sleep(time.Millisecond)
        }
    }()
}

for i := 0; i < 10; i++ {
    go func() {
        writeResp := make(chan bool)
        for {
            writes <- writeOp{key: rand.Intn(5), value: rand.Intn(100), resp: writeResp}
            <- writeResp
            atomic.AddUint64(&writeCnt, 1)
            time.Sleep(time.Millisecond)
        }
    }()
}

time.Sleep(time.Second)
fmt.Println("readCnts: ", atomic.LoadUint64(&readCnt))
fmt.Println("writeCnts: ", atomic.LoadUint64(&writeCnt))
```

## module-and-package

参考[how-to-write-go-code](https://golang.org/doc/code.html)

go 代码的组织关系按照：repo --> module --> package 的方式进行组织管理。在代码的 repo 根目录下需要配置`go.mod`，记录当前库的前缀：

```go
module github.com/whiledong/test
```

这样子，等于在该 repo 中写的代码都有这个全局的前缀限定符。在 repo 中建立的每一个目录或者子目录，都是在该 module 前缀后扩展。go 的目录名称要和 package 的名称一致。import 代码的级别是 package，而项目输出的级别是 module。

如果使用`go install`命令，安装的执行程序放在`$GOPATH/bin/test`中，等于 module 限定符的最后一部分就是程序级别的输出名字。

对于 go 而言，package 的名称和 package 所在的目录名，基本上，除了是 main 的 package，别的情况下都最好要一致：

- go 使用路径名进行 import 的导入
- 导入后，使用对应路径下的 package 名称做为导入的包前缀

所以如果不一致，就会发现导入的路径名称和使用的包名称不一致，比较奇怪。[everything-you-need-to-know-about-packages-in-go](https://medium.com/rungo/everything-you-need-to-know-about-packages-in-go-b8bac62b74cc)

## effective-go

记录[effective-go](https://golang.org/doc/effective_go.html)的学习内容。

- 在 package 定义之前的是包级别的注释，一般定义为 `Package xxxx implements xxxx`

- 对外暴露的接口，一般注释第一个单词就是方法名称，比如：

  ```go
  // Compile parses a regular expression and returns, if successful,
  // a Regexp that can be used to match against text.
  func Compile(str string) (*Regexp, error) {
  ```

  这样子对 grep 比较友好，搜索关键字，就知道第一个单词对应方法名。

- package 的名称，在 go 中提倡更加简单，简洁的方式。`Long names don't automatically make things more readable. A helpful doc comment can often be more valuable than an extra long name.`

- > Go has no comma operator and ++ and -- are statements not expressions. Thus if you want to run multiple variables in a for you should use parallel assignment (although that precludes ++ and --).

  在 for 语句中，如果需要同时操作多个数据的变更，使用 tuple 的变更方式：

  ```go
  // Reverse a
  for i, j := 0, len(a)-1; i < j; i, j = i+1, j-1 {
      a[i], a[j] = a[j], a[i]
  }
  ```

- go 中支持，返回的参数带名称，和入参一样，带名称的参数会初始化为 zero values of type. 如果 return 没有加入参数，会返回命名返回参数的当前数值。

  ```go
  func ReadFull(r Reader, buf []byte) (n int, err error) {
      for len(buf) > 0 && err == nil {
          var nr int
          nr, err = r.Read(buf)
          n += nr
          buf = buf[nr:]
      }
      return
  }
  ```

- go 的 defer 语句，会在调用 defer 之时，就会计算 defer 函数绑定的参数内容。和一般语言的闭包延迟解析机制不太一样，应该算是避免了一种可能的语言坑。

  ```go
  // LIFO: last in first out, output: 4 3 2 1
  for i := 0; i < 5; i++ {
      defer fmt.Printf("%d ", i)
  }
  ```

  defer 的参数，在 defer 调用时解析，利用好可以简化代码逻辑，比如文档中给出的：

  ```go
  /*
  entering: b
  in b
  entering: a
  in a
  leaving: a
  leaving: b
  */
  func trace(s string) string {
      fmt.Println("entering:", s)
      return s
  }

  func un(s string) {
      fmt.Println("leaving:", s)
  }

  func a() {
      defer un(trace("a"))
      fmt.Println("in a")
  }

  func b() {
      defer un(trace("b"))
      fmt.Println("in b")
      a()
  }

  func main() {
      b()
  }
  ```

  `trace`在调用 defer 的过程中，起到了初始化的作用，一行代码做到了 context 的开始和运行 log 的监控。

- `slice`的本质其实是包含了数据指针的结构体，使用**值传递**，但是在修改 slice 内容的时候，会修改到内部的数据。所以，我们可以用`slice`做为入参时，可以修改实际的数据，比如`File.Read`的定义`func (f *File) Read(buf []byte) (n int, err error)`。

- `2D-slice`的分配有两种方式（体现了 go 的灵活），一种是数据可变长的，每次分配一个新的行数据；另外是类似 C 的方式，二维的数组数据本身就是一个一维数组，这样子内存效率更高：

  ```go
  // Allocate the top-level slice.
  picture := make([][]uint8, YSize) // One row per unit of y.
  // Loop over the rows, allocating the slice for each row.
  for i := range picture {
      picture[i] = make([]uint8, XSize)
  }

  // 将行数据指向一个数据段分片
  // Allocate the top-level slice, the same as before.
  picture := make([][]uint8, YSize) // One row per unit of y.
  // Allocate one large slice to hold all the pixels.
  pixels := make([]uint8, XSize*YSize) // Has type []uint8 even though picture is [][]uint8.
  // Loop over the rows, slicing each row from the front of the remaining pixels slice.
  for i := range picture {
      picture[i], pixels = pixels[:XSize], pixels[XSize:]
  }
  ```

- 使用`String() string`接口时，需要留意不要产生类型数据的循环解析：

  ```go
  type MyString string

  func (m MyString) String() string {
      return fmt.Sprintf("MyString=%s", m) // Error: will recur forever.
  }
  ```

  解决的方法很简单，将数据强制转换为基本类型：

  ```go
  type MyString string
  func (m MyString) String() string {
      return fmt.Sprintf("MyString=%s", string(m)) // OK: note conversion.
  }
  ```

- go 中没有继承的概念，继承通过`embedding`来实现。所谓`embedding`就是直接将**父类的方法和字段变成自己的方法和字段**，提供了一种更加简洁（接口）组合方式：

  ```go
  // io.ReadWrite就是一个接口的组合，直接包含了Reader/Write的接口方法
  // ReadWriter stores pointers to a Reader and a Writer.
  // It implements io.ReadWriter.
  type ReadWriter struct {
      *Reader  // *bufio.Reader
      *Writer  // *bufio.Writer
  }
  ```

  注意，`embedding`不需要制定**变量名称**，如果制定了，就不是嵌入，而是定义成员变量，这样子还需要自己定义相关的方法实现，才算是有对应的接口：

  ```go
  type ReadWriter struct {
      reader *Reader
      writer *Writer
  }

  // 还需要自己重新实现对应的接口方法
  func (rw *ReadWriter) Read(p []byte) (n int, err error) {
      return rw.reader.Read(p)
  }
  ```

  对于`embedding`而言，其和继承不同的地方就在于，调用`embedding`类型的方法时，实际是一种**组合的关系**，会将方法委托到对应的实例方法上，就类似上面的代码实现。

  同时我们可以在构造时，制定组合的对象：

  ```go
  type Job struct {
      Command string
      *log.Logger
  }

  func NewJob(command string, logger *log.Logger) *Job {
      return &Job{command, logger}
  }

  // or with a composite literal
  // job := &Job{command, log.New(os.Stderr, "Job: ", log.Ldate)}
  ```

  可以使用**最里面的类名称**做为实际的组合对象进行返回，比如：

  ```go
  func (job *Job) Printf(format string, args ...interface{}) {
      job.Logger.Printf("%q: %s", job.Command, fmt.Sprintf(format, args...))
  }
  ```

  如果嵌入的类名称或者相同层级的字段相同的话，**只要外围不直接使用冲突的名称**，系统不会报错，因为只需要扩展方法和属性而已，否则会报错。

  > if the same name appears at the same nesting level, it is usually an error; it would be erroneous to embed log.Logger if the Job struct contained another field or method called Logger. However, if the duplicate name is never mentioned in the program outside the type definition, it is OK. This qualification provides some protection against changes made to types embedded from outside; there is no problem if a field is added that conflicts with another field in another subtype if neither field is ever used.

- go 的并发设计哲学：

  > Do not communicate by sharing memory; instead, share memory by communicating.

  go 强大的 concurrent 的工具，天然可以用作*生产者-消费者模式*的处理队列，比如文档中提到的 `A Leaky Buffer` 示例。

  该例是 RPC 框架的一个抽象，客户端（这里类 producer）不停读取网络数据，获取到数据后，放入有界空闲队列中，起到了缓存池的作用。处理完数据后，放入空闲池中，等待一个服务端（这里类 consumer）来消费，使用一个无缓存的 channel 进行空闲 Buffer 的传递（个人理解：如果处理方比较繁忙，生产方可以直接休息，而不用接受更多的生产需求，所以使用无缓存的 channel 在这里有这样一层控制语义）

  ```go
  var freeList = make(chan *Buffer, 100)
  var serverChan = make(chan *Buffer)

  func client() {
      for {
          var b *Buffer
          // Grab a buffer if available; allocate if not.
          select {
          case b = <-freeList:
              // Got one; nothing more to do.
          default:
              // None free, so allocate a new one.
              b = new(Buffer)
          }
          load(b)              // Read next message from the net.

          // 无缓存channel，也可能能有多个协程等待，只是只能一个被唤醒
          serverChan <- b      // Send to server.
      }
  }
  ```

  ```go
  func server() {
      for {
          b := <-serverChan    // Wait for work.
          process(b)
          // Reuse buffer if there's room.
          select {
          case freeList <- b:
              // Buffer on free list; nothing more to do.
          default:
              // Free list full, just carry on.
              // 这里可能存在，存在满池的情况：server处理非常满，client处理很快，
              // 又将数据填满freeList的buffer，上一轮处理的数据就放不回去了
          }
      }
  }
  ```

- `panic`不要轻易使用，更多的使用 `error` 进行错误的处理。一种使用 `panic` 的场景是用在 `init` 函数中，如果初始化时候数据状态不对，直接退出程序是一个不错的选择：

  ```go
  var user = os.Getenv("USER")

  func init() {
      if user == "" {
          panic("no value for $USER")
      }
  }
  ```

- 关于*interface-and-methods*，文档中的例子非常生动：

  在 go 中，任何类型都可以绑定方法，所以任何的东西在 go 中都可以满足接口的要求，比如 `http.Handler` 接口：

  ```go
  type Handler interface {
      ServeHTTP(ResponseWriter, *Request)
  }
  ```

  这里，`ResponseWriter` 是一个接口，实现了 `Write` 方法，一般接口在 go 中都直接**使用值类型**；而 `Request` 是一个结构体，所以这里使用指针类型。

  如果需要保存状态，比如容易想到，使用结构体，定义状态数据，并实现接口方法：

  ```go
  // Simple counter server.
  type Counter struct {
      n int
  }

  func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
      ctr.n++
      fmt.Fprintf(w, "counter = %d\n", ctr.n)
  }
  ```

  但实际上，这里其实直接用 int 就可以表示 `Counter` 类型：

  ```go
  // 直接int表示类型，实现对应方法，很有意思
  // int本身就记录了自身的状态
  type Counter int

  func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
      *ctr++
      fmt.Fprintf(w, "counter = %d\n", *ctr)
  }
  ```

  如果需要访问网页的时候，存在一些通知事件，可以将 channel 直接作为类型定义：

  ```go
  // A channel that sends a notification on each visit.
  // (Probably want the channel to be buffered.)
  type Chan chan *http.Request

  func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
      ch <- req
      fmt.Fprint(w, "notification sent")
  }
  ```

  最后，如果我们打算将符合签名的原始方法，转化为无状态的实现 `ServeHTTP` 的类型，可以直接将 **函数** 定义为一个类型，调用该类型就等于进行函数的强制转换：

  ```go
  // The HandlerFunc type is an adapter to allow the use of
  // ordinary functions as HTTP handlers.  If f is a function
  // with the appropriate signature, HandlerFunc(f) is a
  // Handler object that calls f.
  type HandlerFunc func(ResponseWriter, *Request)

  // ServeHTTP calls f(w, req).
  func (f HandlerFunc) ServeHTTP(w ResponseWriter, req *Request) {
      f(w, req)
  }

  // Argument server.
  func ArgServer(w http.ResponseWriter, req *http.Request) {
      fmt.Fprintln(w, os.Args)
  }

  http.Handle("/args", http.HandlerFunc(ArgServer))
  ```

  总结起来：

  1. go 中接口就是方法的集合
  2. 几乎 go 中任何元素都可以定义为一个 type
  3. type 不一定只能用 struct 来包含状态，元素本身就可以作为 type 的状态
  4. type 本质上是**嫁接数据和接口的桥梁**

## book-gopl

记录学习[go-programming-language](https://github.com/adonovan/gopl.io/)一书的笔记。

### defer-return

defer 在 return 之后执行，所以可以用来 debug 返回值信息，甚至是改变返回值数据。

```go
// 返回信息
func double(x int) (result int) {
    // debug result value
    defer func() {fmt.Printf("double(%d)=%d)", x, result}()
    return x+x
}

// 改变信息
func trip(x int) (result int) {
    // hook result
    defer func() { result += x }()
    return double(x)
}
```

### defer-in-for-loop

defer 是在**其定义的函数执行结束**之后执行，如果在 for 循环中多次调用 defer，并不会在下一次循环执行时调用 defer，而是推迟到函数执行之后：

```go
for _, filename : range filenames {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close() // NOTE: risky; could run out of file
    // do something
}
```

上面的代码如果有非常多文件打开，可能导致文件描述符被好耗尽。一个解决方法时将循环体中的 defer 放到另一个函数中：

```go
for _, filename : range filenames {
    if err := doFile(filename); err != nil {
        return err
    }
}

func doFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    // do something
}
```

### anynomous-struct

go 中支持定义匿名的结构，一方面可以使用 struct 的字段来组合数据，类似于一个动态的 json-like-object；另一方面，利用内嵌的方式，匿名的 struct 也可以有对应的类方法。

考虑一个 cache 的实现，用两个包级别的变量来定义 cache 和对应的锁：

```go
var (
    mu sync.Mutex   // guards mapping
    mapping = make(map[string]string)
)

func LookUp(key string) string {
    mu.Lock()
    v := mapping[key]
    mu.Unlock()
    return v
}
```

用匿名 struct 可以改写成为：

```go
cache := struct {
    // 内嵌了Mutex的方法
    sync.Mutex
    mapping map[string]string
}{
    mapping: make(map[string]string)
}

func LookUp(key string) string {
    cache.Lock()
    v := cache.mapping[key]
    cache.Unlock()
    return v
}
```

### nil-interface-vs-nil-interface-value

在 go 中，interface 其实是一个(type, value)的二元组，如果有一个 T 类型的 nil 值給 interface 赋值，对应的 interface 不是 nil，而时 type 为 T，但 value 为 nil，单独用 nil 判断 interface 会有 panic 的风险：

```go
func f(out io.Writer) {
    if out != nil {
        out.Write([]byte("hello world"))
    }
}

func main() {
    var buf *bytes.Buffer
    f(buf)  // NOTE: subtly incorrect!
}
```

nil 可做为类型 T 的有效接收者，比如 `*os.File`，接收 nil 后依然可以工作，但对于 `*bytes.Buffer` 不符合要求，如果用 nil 调用 `Write` 方法，违反了其接口协议：接受者必须非空。

一个解决方法是，统一使用接口类型，而不是具体的实现类型：

```go
func f(out io.Writer) {
    if out != nil {
        out.Write([]byte("hello world"))
    }
}

func main() {
    var buf io.Writer // = new(bytes.Buffer)
    f(buf)
}
```

或者利用反射，检测接口的 value 是不是 nil，参考[这里](https://medium.com/@mangatmodi/go-check-nil-interface-the-right-way-d142776edef1)

```go
// 这里如果i的接口value是一个值类型，调用IsNil会报错
func isNil(i interface{}) bool {
    return i == nil || reflect.ValueOf(i).IsNil()
}

// 先判断i的类型，再调用，更加安全
func isNilFixed(i interface{}) bool {
    if i == nil {
        return true
    }
    switch reflect.TypeOf(i).Kind() {
    case reflect.Ptr, reflect.Map, reflect.Array, reflect.Chan, reflect.Slice:
        return reflect.ValueOf(i).IsNil()
    }
    return false
}
```

### switch-type

switch语句会隐式的创建一个语言块，所以，在switch中的变量可以被重新定义，并且覆盖外面的变量：

```go
func sqlQuote(x interface{}) string {
    // 惯用方法，使用相同的名字重新赋值
    switch x := x.(type) {
    case nil:
        return "NULL"
    case int, uint:
        return fmt.Sprintf("%d", x) // x has type interface{} here.
    case bool:
        if x {
            return "TRUE"
        }
        return "FALSE"
    case string:
        return sqlQuoteString(x) // (not shown)
    default:
        panic(fmt.Sprintf("unexpected type %T: %v", x, x))
    }
}
```

### about-design-interface

> 当设计一个新的包时，新的Go程序员总是通过创建一个接口的集合开始和后面定义满足它们的具体类型。这种方式的结果就是有很多的接口，它们中的每一个仅只有一个实现。不要再这么做了。这种接口是不必要的抽象；它们也有一个运行时损耗。接口只有当有两个或两个以上的具体类型必须以相同的方式进行处理时才需要。
>
> 当一个接口只被一个单一的具体类型实现时有一个例外，就是由于它的依赖，这个具体类型不能和这个接口存在在一个相同的包中。这种情况下，一个接口是解耦这两个包的一个好方式。
>
> 因为在Go语言中只有当两个或更多的类型实现一个接口时才使用接口，它们会从特定的实现细节中抽象出来。结果就是有更少和更简单方法（和io.Writer或fmt.Stringer一样只有一个）的更小的接口。当新的类型出现时，小的接口更容易满足。对于接口设计的一个好标准就是 **ask only for what you need（只考虑你需要的东西）**

### concurrent-memo

使用go的同步原语实现并发缓存结构，具体[参考](https://books.studygolang.com/gopl-zh/ch9/ch9-07.html)

书中提到了2种方式，一种基于加锁，其思想和java的并发编程非常类似：

- 使用future隔离结果和调用协程
- future对象加入通道进行协程同步

自己实现了一个版本，和书中的稍微有些区别：使用读写锁加速效率，毕竟对于读多写少的缓存而言，将读写加锁分开，可以有效提高并发度：

```go
type Func func(string) (interface{}, error)

type future struct {
    v     interface{}
    e     error
    ready chan struct{}
}

type Memo struct {
    f Func
    l sync.RWMutex
    m map[string]*future
}

func New(f Func) *Memo {
    return &Memo{f: f, m: make(map[string]*future)}
}

func (m *Memo) Get(key string) (interface{}, error) {
    // 1. 先用读锁获取数据
    m.l.RLock()
    f := m.m[key]
    m.l.RUnlock()

    // 2. 对于value为指针数据，直接判断nil判断存在性
    if f == nil {

        // 3. 加写锁，再读取一次，保障一定只有一次进入set语义
        m.l.Lock()
        newf := m.m[key]
        if newf == nil {
            f = &future{ready: make(chan struct{})}
            f.v, f.e = m.f(key)
            m.m[key] = f

            // 4. 利用close进行协程同步
            close(f.ready)
        } else {
            f = newf
        }
        m.l.Unlock()
    }

    // 5. 等待ready，close之后的channel，会直接返回
    <-f.ready
    return f.v, f.e
}
```

另外一种就是基于go的并发控制哲学：**share memory by communicating**

- 所有读写缓存请求代理到唯一控制的主协程
- 主协程控制所有cache的读和写
- 请求中加入channel进行流程同步
- 主协程一定不能有太重的阻塞操作，将操作都异步化。

[代码参考](https://github.com/adonovan/gopl.io/tree/master/ch9/memo5)

```go
// Func is the type of the function to memoize.
type Func func(key string) (interface{}, error)

// A result is the result of calling a Func.
type result struct {
    value interface{}
    err   error
}

type entry struct {
    res   result
    ready chan struct{} // closed when res is ready
}

// A request is a message requesting that the Func be applied to key.
type request struct {
    key      string
    response chan<- result // the client wants a single result
}

type Memo struct{ requests chan request }

// New returns a memoization of f.  Clients must subsequently call Close.
func New(f Func) *Memo {
    memo := &Memo{requests: make(chan request)}
    go memo.server(f)
    return memo
}

func (memo *Memo) Get(key string) (interface{}, error) {
    // 请求都通过带有channel的请求和主协程交互
    response := make(chan result)
    memo.requests <- request{key, response}
    res := <-response
    return res.value, res.err
}

func (memo *Memo) Close() { close(memo.requests) }

//!-get

//!+monitor

func (memo *Memo) server(f Func) {
    cache := make(map[string]*entry)
    for req := range memo.requests {
        e := cache[req.key]
        if e == nil {
            // This is the first request for this key.
            e = &entry{ready: make(chan struct{})}
            cache[req.key] = e
            go e.call(f, req.key) // call f(key)
        }
        // 阻塞都代理到别的协程中，保障主协程高速运行
        go e.deliver(req.response)
    }
}

func (e *entry) call(f Func, key string) {
    // Evaluate the function.
    e.res.value, e.res.err = f(key)
    // Broadcast the ready condition.
    close(e.ready)
}

func (e *entry) deliver(response chan<- result) {
    // Wait for the ready condition.
    <-e.ready
    // Send the result to the client.
    response <- e.res
}
```

## misc

### slices-of-interfaces

go 中为什么，`[]T`的 slice 不可以强转换为`[]interface{}`：因为`interface{}`对象，实际上包含 2 个信息，一个是数据本身，一个是具体的类型（这样子才有运行时的反射和动态类型解析）。但普通类型`T`的对象，是不需要动态类型解析的，其类型只存在于编译期。

所以，在 go 中如果这样子转换，需要`O(n)`时间复杂度，而在 go 的设计哲学中，**语法是不能够隐藏复杂度**，所以需要手动实现，利用反射的一个实现代码：

```go
func InterfaceSlice(slice interface{}) []interface{} {
    s := reflect.ValueOf(slice)
    if s.Kind() != reflect.Slice {
        panic("InterfaceSlice() given a non-slice type")
    }

    ret := make([]interface{}, s.Len())

    for i:=0; i<s.Len(); i++ {
        ret[i] = s.Index(i).Interface()
    }

    return ret
}
```

- 具体参考 stackoverflow 的回答[type-converting-slices-of-interfaces](https://stackoverflow.com/questions/12753805/type-converting-slices-of-interfaces)
- golang 官方的说明[go-wiki-interface-slice](https://github.com/golang/go/wiki/InterfaceSlice)
- 非常好的介绍 interface 的疑点[how-to-use-interfaces-in-go](https://jordanorelli.com/post/32665860244/how-to-use-interfaces-in-go)

### formatting

go 中几个有用的，和别的编程语言不太一样的格式控制符：

- `%v`：打印 go 类型的基本表示。
- `%+v`：打印 struct 时，加入 filed 名称。
- `%#v`：打印 struct 时，同时加入 struct 的名称和 filed 名称。
- `%T`：打印类型

### goproxy

大陆的官方本地代理服务，go 的 package 挂载 CDN，具体参考项目主页[goproxy.cn](https://github.com/goproxy/goproxy.cn)

在 mac 中需要配置环境变量：

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.cn
```

### switch中设置timer

考虑这样的场景，设置一个1s的ticker，和一个2s的stop timer。如果写成下面的样子，stop timer永远不会执行：

```go
    for {
        select {
        case <-time.After(2*time.Second):
            goto exit
        case t := <-time.Tick(time.Second)
            fmt.Printf("time meet at %v\n", t)
        }
    }

exit:
    fmt.Printf("exit")
```

因为在select逻辑中，每次select执行都会**重新构造case的运行结构体，对应的channel也会重新构造**，所以，在select中构造的channel，在
重新执行select的时候，又会重新构造。

改成下面的形式也不对：

```go
    for {
        // 治标不治本，到了下一次for的时候，重新构造的timer，存在同样的问题。设定时间小的先执行，且永远先执行
        afterTimeChannel := time.After(2 * time.Second)
        tickTimeChannel := time.Tick(time.Second)

        select {
        case <-afterTimeChannel:
            goto exit
        case t := <-tickTimeChannel:
            fmt.Printf("time meet at %v\n", t)
        }
    }

exit:
    fmt.Printf("exit")
```

需要将timer的构造放在更全局的位置：

```go
    afterTimeChannel := time.After(2 * time.Second)
    tickTimeChannel := time.Tick(time.Second)

    // 理解：for和select是一个统一逻辑体，timer放在for的外面
    for {
        select {
        case <-afterTimeChannel:
            goto exit
        case t := <-tickTimeChannel:
            fmt.Printf("time meet at %v\n", t)
        }
    }

exit:
    fmt.Printf("exit")
```

参考：

- [golang-select](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-select/)
- [stackoverflow-timer-in-select-never-meet](https://stackoverflow.com/questions/37691887/why-does-time-after-never-fire-when-it-is-paired-with-a-ticker-in-a-select-block/62587485#62587485)

## go-land 技巧

go-land有几个非常好用的技巧：

- 使用`Extract Variable`，提取当前的上下文内容为一个变量。
- 使用`Inline Variable`，将当前变量嵌入到上下文中。
- completion时候，使用tab可以替换当前的内容，比如将printf替换到println，而使用回车不会替换，而是插入。
- 连续两次使用completion按键，会提示可将当前上下文作为第一个变量的函数列表。
- postfix completion，非常有意思，使用`rr`表示`return error`，在一个函数返回error的情况下，改成如果发现`err != nil`返回error的语句。
