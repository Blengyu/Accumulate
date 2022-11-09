# Golang的关键字

## defer：将函数推迟到外层函数返回之后执行

### 规则一：延迟函数的参数在defer语句出现时就已经确定下来了

```go
func` `a() {  i := 0
  ``defer` `fmt.Println(i)
  ``i++
  ``return
}
```

结果仍打印0 注意：return不是原子操作，执行过程是: 保存返回值(若有)-->执行defer（若有）-->执行return跳转

### 规则二：延迟函数执行按后进先出顺序执行，即先出现的defer最后执行

定义defer类似于入栈操作，执行defer类似于出栈操作；每申请到一个用完需要释放的资源时，立即定义一个defer来释放资源是个很好的习惯

### 规则三：延迟函数可能操作主函数的具名返回值

```
func` `deferFuncReturn() (result int) {  i := 1
 ``defer` `func``() {
    ``result++
  ``}()  ``return` `i}
```

```go
result = i
result++
return
```

return之后的语句先执行，defer后的语句后执行 。即先执行result=i 再执行defer的result++ 最后执行汇编指令ret

## Recover

1.recover必须定义在defer中，并且在panic之前，为panic异常兜底，接收panic异常消息，并恢复程序执行

2.即使使用recover恢复程序，panic所在方法后面的程序不会执行

3.子协程发生panic(),主线程不能recover()

当 panic() 触发程序退出发生时，panic() 后面的代码将不会被运行（如果在某个方法，在该方法后面的不会执行，调用该方法的程序仍然可以执行），但是在 panic() 函数前面已经运行过的 defer 语句依然会在宕机发生时发生作用，所以recover必须定义在defer 主体中，为panic兜底。


## 控制并发顺序

### 1.GMP模型

G：Goroutine 我们所说的协程，为用户级的轻量级线程，每个Goroutine对象中的sched保存着其上下文信息

goroutine的优点：

1.占用的内存更小(几kb)
初始为2kb，如果栈空间不足则自动扩容
2.调度更灵活(runtime调度)
Go自己实现的调度器，创建和销毁的消耗非常小，是用户级。
3.抢占式调度(10ms)
编译器插入抢占指令，函数调用时检查当前Goroutine是否发起抢占请求
4.1.14版本后支持基于信号的异步抢占(20ms)
垃圾回收扫描栈时触发抢占调度
解决抢占式调度因垃圾回收和循环长时间占用资源（无法执行抢占指令）导致程序暂停

P：Processor 调度，即为G和M的调度对象，用来调度G和M之间的关联关系，P与M建立连接后，使P中可运行的G获得运行时机并执行，其数量可通过GOMAXPROCS()来设置，默认为核心数。P里面一般会存当前goroutine运行的上下文环境（函数指针，堆栈地址及地址边界

当直接从P本地的G列表，全局G列表获取不到G时，会从netpoller处获取G，此时获取不到G，才会从其他P的本地队列获取一半的G。

P找不到G就进入第二阶段 对P进行处理

M：Machine 真正的工人，对内核级线程的封装，数量对应真实的CPU数

M的数量和P不一定匹配，可以设置很多M，M和P绑定后才可运行，多余的M处于休眠状态

![1](E:\Accumulate\golang\1.png)

#### 1.1context上下文

context类型用来简化对于处理单个请求的多个 `goroutine` 之间与请求域的数据、取消信号、截止时间等相关操作，这些操作可能涉及多个 [API](https://so.csdn.net/so/search?q=API&spm=1001.2101.3001.7020) 调用

如果所依赖的API调用缓慢，你不希望增加负载并且降低所有请求执行效率，可以使用超时或ddl-context

```go
ctx := context.TODO()  创建一个空context，静态分析工具可以使用它来验证 context 是否正确传递
```

```go
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(2 * time.Second)) 
ctx := context.WithValue(context.Background(), key, "test")
```

对上下文的理解:点鞭炮，一条引线可以同时点燃左右两串鞭炮，引线又是由一节节火药连成的（链表），突然发现右边鞭炮附近有小孩（在程序中就是遇到了error），要及时停止右边的鞭炮继续燃烧，避免造成不可挽救的情况（在系统中就是资源损耗），这个时候肯定还在继续燃烧对不对（必须对），那其中一节就及时抛出警告，告诉后面的火药别烧下去了（抛出 error ），告诉前面的领导咱们不能再烧了，最后右边的鞭炮就点不起来，也没造成人身伤害。总结下来就是用于取消一个 goroutine，取消多个 goroutine 和传递上下文信息

结合linux的CPU上下文切换进行理解



### 2.select

select 语句被用来执行多个通道操作的一个和其附带的 case 块代码，select 语句将阻塞，因此 select 将等待，直到有 case 语句不阻塞。用协程，通道和 `select` 语句，我们可以向多个服务器请求数据并获取其中最快响应的那个

### 3.sync.WaitGruop

这是一个带着计数器的结构体，这个计数器可以追踪到有多少协程创建，有多少协程完成了其工作。当计数器为 0 的时候说明所有协程都完成了其工作

`Add` 方法的参数是一个变量名叫 delta 的int 类型参数

`Wait` 方法用来阻塞当前协程。一旦计数器为 0, 协程将恢复运行

`Done` 方法可以降低计数器的值，每执行一次减一。

### 4.channel

这是goroutine之间的沟通渠道，通道是有类型的，可以是 int 类型的通道接收整数或错误类型的接收错误等。写入信息：chan<-1    接受信息：var:=<-ch

一个channel通道只能传入数据后必须有协程来接收数据，否则会阻塞，所以引入缓冲区的概念

c := make(chan Type, n)

读缓冲区的操作是渴望式读取，意味着一旦读操作开始它将读取缓冲区所有数据，直到缓冲区为空。（读已经关闭的 chan 能一直读到东西没有就是这个chan类型的零值，比如整型是 int，字符串是 ""写会直接panic）

在缓冲区里面的协程刚好用完的时候是不会阻塞的，主线程就不会等子协程完成，直接结束，通道的长度和容量与切片类似。

## Print

**Println :可以打印出字符串，和变量**

**Printf : 只可以打印出格式化的字符串,可以输出字符串类型的变量，不可以输出整形变量和整形**

Printf整数值：

%b 二进制表示
%c 相应Unicode码点所表示的字符
%d 十进制表示
%o 八进制表示
%q 单引号围绕的字符字面值，由Go语法安全地转义
%x 十六进制表示，字母形式为小写 a-f
%X 十六进制表示，字母形式为大写 A-F
%U Unicode格式：U+1234，等同于 "U+%04X"

浮点数及复数：
%b 无小数部分的，指数为二的幂的科学计数法，与 strconv.FormatFloat中的 'b' 转换格式一致。例如 -123456p-78
%e 科学计数法，例如 -1234.456e+78
%E 科学计数法，例如 -1234.456E+78
%f 有小数点而无指数，例如 123.456
%g 根据情况选择 %e 或 %f 以产生更紧凑的（无末尾的0）输出
%G 根据情况选择 %E 或 %f 以产生更紧凑的（无末尾的0）输出

字符串和bytes的slice表示：
%s 字符串或切片的无解译字节
%q 双引号围绕的字符串，由Go语法安全地转义
%x 十六进制，小写字母，每字节两个字符
%X 十六进制，大写字母，每字节两个字符

**Sprintf：是把格式字符串输出到指定字符串中，所以参数比printf多一个char*。那就是目标字符串地址。（字符串格式化，并把格式化后的字符串返回，所以可以用于赋值操作，终端中不会有显示)，返回为 格式化后的字符串**



**Fprintf() 是把格式字符串输出到指定的文件设备中，所以参数比Printf 多一个文件指针File主要用于文件操作，Fprintf() 是格式化输出到一个 Stream ,通常是一个文件**



     %v 输出结构体 {10 30}
     
    %+v 输出结构体显示字段名 {one:10 tow:30}
    
    %#v 输出结构体源代码片段 main.Point{one:10, tow:30}
    
    %T 输出值的类型            main.Point
    
    %t 输出格式化布尔值      true
    
    %d`输出标准的十进制格式化 100
    
    %b`输出标准的二进制格式化 99 对应 1100011
    
    %c`输出定整数的对应字符  99 对应 c
    
    %x`输出十六进制编码  99 对应 63
    
    %f`输出十进制格式化  99 对应 63
    
    %e`输出科学技科学记数法表示形式  123400000.0 对应 1.234000e+08
    
    %E`输出科学技科学记数法表示形式  123400000.0 对应 1.234000e+08
    
    %s 进行基本的字符串输出   "\"string\""  对应 "string"
    
    %q 源代码中那样带有双引号的输出   "\"string\""  对应 "\"string\""
    
    %p 输出一个指针的值   &jgt 对应 0xc00004a090
    
    % 后面使用数字来控制输出宽度 默认结果使用右对齐并且通过空格来填充空白部分
    
    %2.2f  指定浮点型的输出宽度 1.2 对应  1.20

## 指针

```
func TestSwitch(t *testing.T) {
	b := 255
	a := &b
	fmt.Printf("Type of a is %T\n", a)
	fmt.Println("address of b is", &b)
	fmt.Println("value of a is", a)
	fmt.Println("============", &a)
	fmt.Println("============", *a)
}
```

结果：

Type of a is *int                          a的类型是指针int类型
address of b is 0xc00009e3e0  b的地址
value of a is 0xc00009e3e0        a存的值是b的地址
============ 0xc0000c4030  变量a自身的地址（与b不同）
============ 255                      指针a指向地址的值

## int uint的区别

```go
fmt.Println("不同int类型占用的字节数大小：")
var i1 uint = 1
var i2 uint8 = 2
var i3 uint16 = 3
var i4 uint32 = 4
var i5 uint64 = 5
fmt.Printf("uint    : %v\n", unsafe.Sizeof(i1))
fmt.Printf("uint8   : %v\n", unsafe.Sizeof(i2))
fmt.Printf("uint16  : %v\n", unsafe.Sizeof(i3))
fmt.Printf("uint32  : %v\n", unsafe.Sizeof(i4))
fmt.Printf("uint64  : %v\n", unsafe.Sizeof(i5))
```

结果：

不同uint类型占用的字节数大小（int相同）：                     uint类型的取值范围：  int类型的取值范围：

uint    : 8                                                                                   0~2^64                         -2^63~2^63
uint8   : 1                                                                                  0~2^8                           -2^7~2^7
uint16  : 2                                                                                 0~2^16                         -2^15~2^15
uint32  : 4                                                                                 0~2^32                         -2^31~2^31
uint64  : 8                                                                                 0~2^64                         -2^63~2^63
