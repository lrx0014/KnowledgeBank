# Golang 协程正确的使用方法

This page updated at: **2019/03/18**

## 错误的使用方法

```golang
package main
// 错误使用案例
import (
    "time"
    "fmt"
)
var c1 chan string = make(chan string)
func main(){
    func(){
       time.Sleep(time.Second * 2)
        c1 <- "result 1"
    }()
    fmt.Println("c1 is", <-c1)
}
```

由于 c1 在创建时没有指定缓存长度, 所以在写入channel c1 时，必须应该有对应的接收端在等待，上例中，由于对c1的写入时c1没有缓存区且没有接收端，所以就报错了。

```golang
fmt.Println("c1 is", <-c1)
```

这段代码本来是可以阻塞的，但是这种用法是错误的

## 正确的使用方法 （将写入放到单独的协程中）

通过将写入端放到单独的协程中，使得channel 的写入端和接收端可以对接。

```golang
func main(){
    go func(){
       time.Sleep(time.Second * 2)
        c1 <- "result 1"
    }()
    fmt.Println("I am here")
    fmt.Println("c1 is", <-c1)
}
```

可以看到 匿名函数部分代码是在独立的协程中执行的，在同一个协程中对一个channel写入时，如果channel 没有缓存长度，必然会报错，见错误案例

## select 用法

select 可以同时监听多个channel

### 阻塞方式

执行到select 语句时如果所有的case 中的channel 都没有数据，select 语句将会阻塞，直到一个case中的 channel 获取到数据。

```golang
package main
import (
    "time"
    "fmt"
)

func AskX(bid1 chan int) {
    for i:=100 ;i<105 ; i++{
        time.Sleep(time.Second*1)
        bid1 <- i
    }
}
func AskY(bid2 chan int) {
    for i:=0 ; i<5; i++{
        time.Sleep(time.Second*2)
        bid2 <- i
    }

}

func main(){
    bid1 := make(chan int)
    bid2 := make(chan int)

    go AskX(bid1)
    go AskY(bid2)

    select {
    case Xbid := <-bid1:
        fmt.Println(Xbid)
    case Ybid := <-bid2:
        fmt.Println(Ybid)
    }
}
```

运行的结果是，代码被阻塞1秒后输出100, 因为channel bid1 在1秒后拿到了数据，select 阻塞结束输出

### 非阻塞方式

将阻塞部分 select 处代码修改为

```golang
select {
    case Xbid := <-bid1:
        fmt.Println(Xbid)
    case Ybid := <-bid2:
        fmt.Println(Ybid)
    default :
        fmt.Println("no data.")
    }
```

可以看到只添加了 default 语句，表示当所有的 channel 都取不到数据时，默认执行 default 后面的语句

### 带超时机制的

```golang
select {
   case Xbid := <-bid1:
       fmt.Println(Xbid)
   case Ybid := <-bid2:
       fmt.Println(Ybid)
   case <-time.After(time.Second*2):
       fmt.Println("overtime for 2 seconds")
   }
```

上面的代码表示，当所有channel 超过2秒还没有取到数据，就会执行超时设置部分的代码。

## 总结

* 在同一个协程中如果对一个channel写入数据，如果channel 没有缓存长度，必然报错

* 在主协程中有接收端，在子协程中没有发送端会报错

* 在子协程中有发送端，在其他协程中没有接收端，不会报错
