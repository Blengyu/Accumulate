# go面试

## go语言类问题

GMP模型是如何实现的

**解答：G是一个goroutine对象，每次go调用的时候都会创建一个G对象；M就是Machine，代表真实的处理器核心数量，也是真正执行G的计算资源；P是Processor，负责调度G和M的执行，G依赖于P，P必须与M进行绑定，且GOMAXPROCS()来设置数量** 

  进程，线程，协程联系和区别 

**解答：联系----系统中所有的应用程序都以进程的方式运行（任务管理器结束进程）；线程是程序执行的基本单元，各个线程之间共享进程的内存空间；协程是轻量级线程。火车看作进程，车厢就是线程，车厢之间可以互通，协程就是去掉多余负重的车厢**

**区别----进程拥有自己的堆栈，不共享堆和栈，是由操作系统进行调度的。**
**线程拥有自己的独立的栈和共享的堆，也是由操作系统进行调度。**
**协程共享堆，不共享栈，协程的调度由用户控制。**

  其他语言有协程吗？

**解答：C#、python、kotlin都有协程**

  一颗CPU，两个协程，其中一个协程在死循环，会发生什么 ？

**解答：go1.14版本以前没有抢占式调度，所以会导致唯一的P线程会不停地工作，另一个协程找不到调度的机会，也就阻塞在了死循环中。go1.14版本之后实现了基于信号的抢占式调度；即抢占运行时间过长（10ms）的G和阻塞在系统调用上的P，然后发送信号给M，M休眠对应的G然后重新调度**

  GC垃圾回收机制和JAVA垃圾回收机制有啥区别 

  Channel底层原理 

  用Channel和两个协程实现数组相加 

  用协程实现顺序打印123 

  切片原理 和数组的区别 

  切片初始化问题 

  map什么内容不能成为key 

  map和sync map（读写问题） 

  看过啥底层包（net，sync等等） 

  懂不懂RPC

  项目怎么实现高并发高性能

make和new的区别，从源码层面讲讲

**解答：**

**1.make仅用于slice（切片）、map（字典）、channel（通道）的内存分配，new可以为所有类型分配内存  **

**原因：因为silice、map、channel底层结构要求在创建的时候必须初始化，不初始化就是nil，如下代码**

**map如果是nil，是不能往map插入元素的，插入元素会引发panic**

**chan如果是nil，往chan发送数据或者从chan接收数据都会阻塞**

 **slice会有点特殊，理论上slice如果是nil，也是没法用的。但是append函数处理了nil slice的情况，可以调用append函数对nil slice做扩容。但是我们使用slice，总是会希望可以自定义长度或者容量，这个时候就需要用到make**

为什么slice是nil也可以直接append？

**解答：对于nil slice，append会对slice的底层数组做扩容，通过调用mallocgc向Go的内存管理器申请内存空间，再赋值给原来的nil slice。**

**2.make的返回值为引用类型本身，不是指针，new的返回值为引用类型的指针**

**3.make在创建时会分配内存并进行初始化对应类型的零值；new只是将内存清零，不会初始化内存**，可能在栈上，也可能在堆上分配内存

**原因：map、slice、channel底层是结构体，需要使用make进行初始化****

**type hchan struct {**
    **qcount   uint           
    dataqsiz uint          
    buf      unsafe.Pointer**
    **elemsize uint16         
    closed   uint32        
    elemtype *_type** 
    **sendx    uint  
    recvx    uint  
    recvq    waitq** 
    **sendq    waitq  
    lock mutex**
**}**

**type hmap struct {**
    **count     int** 
    **flags     uint8**
    **B         uint8  
    noverflow uint16** 
    **hash0     uint32** 
    **buckets    unsafe.Pointer** 
    **oldbuckets unsafe.Pointer** 
    **nevacuate  uintptr      
    extra *mapextra** 
**}**
**type slice struct {    array unsafe.Pointer    len   int    cap   int }**

```go
var` `a *[]int
fmt.Printf(``"a: %p %#v \n"``, &a, a) ``//a: 0xc042004028 (*[]int)(nil)
av := ``new``([]int)
fmt.Printf(``"av: %p %#v \n"``, &av, av) ``//av: 0xc000074018 &[]int(nil)
(*av)[0] = 8
fmt.Printf(``"av: %p %#v \n"``, &av, av) ``//panic: runtime error: index out of range
```

```
var` `m map[string]string
fmt.Printf(``"m: %p %#v \n"``, &m, m)``//m: 0xc042068018 map[string]string(nil) 
mv := ``new``(map[string]string)
fmt.Printf(``"mv: %p %#v \n"``, &mv, mv)``//mv: 0xc000006028 &map[string]string(nil)
(*mv)[``"a"``] = ``"a"
fmt.Printf(``"mv: %p %#v \n"``, &mv, mv)``//这里会报错panic: assignment to entry in nil map
cv := new(chan string)

fmt.Printf("cv: %p %#v \n", &cv, cv)//cv: 0xc000074018 (*chan string)(0xc000074020) 

//cv <- "good" //会报 invalid operation: cv <- "good" (send to non-chan type *chan string)
```

```go
av := make([]int, 5)
fmt.Printf(``"av: %p %#v \n"``, &av, av) ``//av: 0xc000046400 []int{0, 0, 0, 0, 0}
av[0] = 1
fmt.Printf(``"av: %p %#v \n"``, &av, av) ``//av: 0xc000046400 []int{1, 0, 0, 0, 0}
mv := make(map[string]string)
fmt.Printf(``"mv: %p %#v \n"``, &mv, mv) ``//mv: 0xc000074020 map[string]string{}
mv[``"m"``] = ``"m"
fmt.Printf(``"mv: %p %#v \n"``, &mv, mv) ``//mv: 0xc000074020 map[string]string{"m":"m"}
chv := make(chan string)
fmt.Printf(``"chv: %p %#v \n"``, &chv, chv) ``//chv: 0xc000074028 (chan string)(0xc00003e060)
go func(message string) {
  ``chv <- message ``// 存消息
}(``"Ping!"``)
fmt.Println(<-chv) ``// 取消息 //"Ping!"
close(chv)
```

GPM调度

10个goroutine，1个进行死循环，其他的会被调度吗？

slice append()的过程，从源码层面讲

slice 为什么这么扩容，怎么扩容的，从源码层面讲讲扩容

## 算法类问题



链表排序插入，二叉树找中间一段子树（题号437），层次遍历等等L网站初级或中级题目，初级回溯算法
排序算法，堆排序，桶排序，快排，二分查找等等手写，并且举例说出最优和最差情况

## 计算机网络问题

建议玩一天抓包，基本的内容也就熟悉了。 

  HTTP协议报文内容，常见状态码，挂了怎么办。 

  TCP三次握手，四次挥手，以及通信中间挂了怎么办。 

  TCP UDP报文格式以及区别，为啥要那些字段，分别能传输最大报文为多少。 

  OSI7层模型说说每层的常用协议。 

  ICMP,IGMP协议是怎么回事，怎么实现的。 

  为啥要IP还要mac。 

  常见路由协议。 

  ARP协议是怎么回事，报文内容有啥。 

  一个包怎么能从一台电脑到另一台电脑。 

 TCP首部字段有哪些

TCP 可靠的机制

PCB中有哪些信息，详细点

进程间通信的方式，信号量有哪些，信号有哪些

## 数据库问题

ACID
隔离级别
备份还原
redis基本数据类型，RDB和AOF
基础查表建表问题

## 日志级别

1、DEBUG 指定细粒度信息事件是最有用的应用程序调试，主要用于开发过程中打印一些运行信息，一般使用log.debug()进行跟踪调试。

2、INFO 指定能够突出在粗粒度级别的应用程序运行情况的信息的消息，就是用于生产环境中输出程序运行的一些重要信息。info级别监控系统运行情况，可以帮助程序员有效的了解程序的流转。

3、WARN 指定具有潜在危害的情况，暂时可以不处理，但在以后的环境中尽可能去解决，一般很少使用。

4、ERROR  错误事件可能仍然允许应用程序继续运行。就是对单次的请求显示错误信息。比如接口访问超时，用recover() 捕获异常，发生异常的时候global.GVA_LOG.Error("服务器异常!", zap.Any("err", err))输出错误信息，并不影响程序的运行。

5、Fatal 指整个应用程序都会崩溃掉，不能再继续运行错误事件。

如果将log level设置在某一个级别上，那么比此级别优先级高的log都能打印出来。例如，如果设置优先级为WARN，那么OFF、FATAL、ERROR、WARN 4个级别的log能正常输出，DEBUG，INFO会被忽略

总之，使用各种日志库或者第三方的日志库都是为了监视程序再运行过程中可能会出现的问题，应对不同的情况需要不同级别的日志以此来准确高效的定位错误。