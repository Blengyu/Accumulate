# DEX项目

## 项目排错--问题1

问题描述：golang的经典问题，在for循环中使用了goroutine,在goroutine中使用了for循环的参数

```go
wg := new(sync.WaitGroup)
for a, v := range model.PathsAll {
   wg.Add(1)
   go func(wg *sync.WaitGroup) {
      for j, _ := range temp1.Paths {
         temp2 := j
         slippagePrice := PairPrice(slippage, v.Paths[j], client, icount)
```

问题原因：例子中在go func(){}中使用了外层for循环的参数v，所以闭包go协程里面引用的是变量v的地址，值是变化的但地址不变;所有的go协程启动后等待调用，在上面的协程中，部分协程很可能在for循环完成之后才被调用，所以输出结果很多都是最后一次循环的值。

问题本质:golang的for循环会使用同一个变量来存储迭代过程中的临时变量，在将该变量传递给goroutine时，goroutine得到的是该变量的地址，又由于goroutine的启动与调度机制有关，可能for循环执行完后，goroutine才开始调度，所以导致多个goroutine访问的是同一个数据。

**解决方案：使用临时变量tempV:=v然后操作临时变量**

## **项目排错--问题2**

问题描述：调用自己编写的智能合约时，想要执行swapTokenforToken类型的交换函数，出现了转账到合约地址的情况。

问题原因：合约地址是测试网合约的地址，在修改后，没有及时push修改后的代码到gitlab上，再后续其他同事push代码后，没有规范去拉取代码，导致地址修改无效，也就执行了原来的地址。

bug：调用swap交换代币函数 执行transfer函数

**解决方案：修改地址后能正确交换代币，但是对于执行transfer函数的原因未知 另：应该加深异常处理的意识，如：返回nil应该处理异常而不是继续往下走，会给其他模块带来排错的时间成本**

## **项目排错--问题3**

问题描述：调用自己编写的智能合约时，想要把合约里面的所有余额都提出来，出现了USDT无法转出的情况。

问题原因：以太坊上面的USDT不是标准ERC20token，USDT 合约对 transfer 方法的具体实现， 没有返回值

bug：无法把合约里的USDT余额转账交换USDT

**解决方案：全网暂无**