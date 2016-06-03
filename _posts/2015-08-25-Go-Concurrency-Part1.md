---
layout: post
title: "Go Concurrency Part1: Producer and Consumer"  
image:   
categories:
- Golang

tags:
- Concurrency
- Producer
- Consumer

---


## 生产者-消费者模式

[Mexican_Standoff](/assets/images/1280px-Mexican_Standoff.jpg)

生产者-消费者模式即所谓的Fan-in设计模式，这里建立两个channel，一个由多个生产者将生产的数据写入；另外一个由一个消费者读出。

线程同步由主线程借助两个channel完成，其本质是利用了阻塞实现同步。

### 生产者

一个大loop将每个新的i值循环写入。

```
  8 func producer(ch chan int, d time.Duration) {
  9     var i int
 10
 11     for {
 12         ch <- i
 13         i++
 14         fmt.Printf("Producer(GID:%v) i:%v\n", GetGID(), i)
 15         time.Sleep(d)
 16     }
 17 }

```

### 消费者

```
 19 func reader(out chan int) {
 20     for x := range out {
 21         fmt.Printf("Reader(GID:%v) i:%v\n", GetGID(), x)
 22     }
 23 }

```

### 线程同步

从写入channel rang loop的读取，塞入输出channel。

```
 34     //multiplex by main thread range loop
 35     for i := range ch {
 36         out <- i
 37     }

```

这里利用了阻塞channel的特性：如果读取不到则阻塞，可以加入超时。

## 死锁的产生

现象：

```
fatal error: all goroutines are asleep - deadlock!

```

### 死锁原因1：线程都在接收导致的死锁

将生产者的loop变成有限次执行后退出。

```
  8 func producerReceiveDeadLock(ch chan int, d time.Duration) {
  9     var i int
 10
 11     for i < 10 {
 12         ch <- i
 13         i++
 14         fmt.Printf("Producer(GID:%v) i:%v\n", GetGID(), i)
 15         time.Sleep(d)
 16     }
 17 }

```

将会导致main线程和reader线程阻塞:  

```
Producer(GID:18) i:10
Producer(GID:19) i:10
Reader(GID:17) i:9
Reader(GID:17) i:9
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:35 +0xfe

goroutine 17 [chan receive]:
main.reader(0x82026c060)
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:20 +0x72
created by main.main
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:29 +0x7a
exit status 2

```

- Step1: 由于producer线程退出，ch没有人给写入数据，此时为空；
- Step2: 此时线程**goroutine 1**从ch中尝试读取数据，发现读不到，因此阻塞在该channel上,该线程ID为1，即main线程,导致out中无法写入数据，此时为0：

```
 35     for i := range ch {

```
- Step3：而读线程**goroutine 17**不知情(该线程由main创建，ID为17)，仍从out读取数据，导致reader最终也处于了阻塞状态。

```
20     for x := range out {

```

### 死锁原因2：线程都在发送导致的死锁

将消费者行为改为对数据不做消费：

```
 38 //When reader donothing, out channel keep blocking
 39 func readerSendDeadLock(out chan int) {
 40 }

```

导致多个线程都往channel中写数据，这里channel是不一致的（可以一致）：

```

Producer(GID:6) i:1
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan send]:
main.main()
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:54 +0x1ed

goroutine 5 [chan send]:
main.producer(0x82020e0c0, 0x5f5e100)
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:13 +0x7b
created by main.main
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:49 +0x15a

goroutine 6 [chan send]:
main.producer(0x82020e0c0, 0x5f5e100)
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:13 +0x7b
created by main.main
	/Users/fyang/work/concurrency_example/LeakDetect/fanin.go:50 +0x185
exit status 2

```
- Step1: producer生产一个数据，塞入ch；
- Step2：线程**goroutine 1**为main线程， 从ch中拿走一个，往out中写数据：

```
54         out <- i
```

out无人消费，导致写了一个就已经满了，因此阻塞。

- Step3：线程**goroutine 5**为producer1和线程**goroutine 6**此时仍在往ch中写数据：

```
13         ch <- i

```

发现ch数据没有被消费，写不进去而阻塞。

## 如何通知退出循环

Close行为与loop之间通过s.closing

```
type sub struct {
	closing chan chan error	
}

```

- 服务的Loop循环监听该channel的请求
- client往channel发送一个请求：退出和响应该error
- 差点


通过一个channel发送消息给待关闭的thread，将channel类型设置为channel，这样loop可以将错误的原因发送过来:

```

func (s *sub) Close() error {
	errc := make(chan error)	//创建一个无缓冲channel，开始是空的
	s.closing <- errc //向s.closing这个channel发送一个errc(该channel的类型为chanel error)
	return <-errc	//如果没有收到消息，此时close的调用者阻塞在这里，等待从errc中取出该错误类型并返回。
}

```

Loop的处理：取走error并退出

```
var err error //获取失败的时候，设置该error

for {
	select {
	case errc := <-s.closing：	//从中取出一个消息
		errc <- err	//将错误消息塞入该通知性质的channel
		close(s.updates) //告诉receviver我们done
		return
	}
}

```

### select 和 nil channel

对于nil channel，无论发送与接收操作都会阻塞；
select永远不会选择一个阻塞的case.

nil channel与空channle的区别：？

## 参考代码

```

 1 package main
  2
  3 import (
  4     "fmt"
  5     "runtime"
  6     "time"
  7 )
  8
  9 func producer(ch chan int, d time.Duration) {
 10     var i int
 11
 12     for {
 13         ch <- i
 14         i++
 15         fmt.Printf("Producer(GID:%v) i:%v\n", GetGID(), i)
 16         time.Sleep(d)
 17     }
 18 }
 19
 20 //When producer thread quit, ch empty and block readers(reader and main)
 21 func producerReceiveDeadLock(ch chan int, d time.Duration) {
 22     var i int
 23
 24     for i < 10 {
 25         ch <- i
 26         i++
 27         fmt.Printf("Producer(GID:%v) i:%v\n", GetGID(), i)
 28         time.Sleep(d)
 29     }
 30 }
 31
 32 func reader(out chan int) {
 33     for x := range out {
 34         fmt.Printf("Reader(GID:%v) i:%v\n", GetGID(), x)
 35     }
 36 }
 37
38 //When reader donothing, out channel keep blocking
 39 func readerSendDeadLock(out chan int) {
 40 }
 41
 42 //TODO: add timeout for reader
 43 func readerTimeout(out chan int) {
 44     for {
 45         select {
 46         case x := <-out:
 47             fmt.Printf("Reader(GID:%v) i:%v\n", GetGID(), x)
 48         case <-time.After(2 * time.Second):
 49             fmt.Printf("Reader(GID:%v) timeout\n", GetGID())
 50             break
 51         }
 52         fmt.Printf("Reader(GID:%v) will sleep %s and retry\n", GetGID(), 1*time.Second)
 53         time.Sleep(1 * time.Second)
 54     }
 55 }
 56
 57 func main() {
 58     fmt.Println(runtime.NumGoroutine())
 59     ch := make(chan int)
 60     out := make(chan int)
 61     fmt.Printf("InLen:%v OutLen:%v\n", len(ch), len(out))
 62
 63     go reader(out)
 64     //go readerSendDeadLock(out)
 65     //go readerTimeout(out)
 66
 67     go producer(ch, 100*time.Millisecond)
 68     go producer(ch, 100*time.Millisecond)
 69     //go producerReceiveDeadLock(ch, 100*time.Millisecond)
 70     //go producerReceiveDeadLock(ch, 100*time.Millisecond)
 71
 72     //multiplex by main thread range loop
 73     for i := range ch {
 74         out <- i
 75     }
 76
 77     fmt.Println(runtime.NumGoroutine())
 78 }


```

获取goroutine的GID：

```
  1 package main
  2
  3 import (
  4     "bytes"
  5     "runtime"
  6     "strconv"
  7 )
  8
  9 func GetGID() uint64 {
 10     b := make([]byte, 64)
 11     b = b[:runtime.Stack(b, false)]
 12     b = bytes.TrimPrefix(b, []byte("goroutine "))
 13     b = b[:bytes.IndexByte(b, ' ')]
 14     n, _ := strconv.ParseUint(string(b), 10, 64)
 15     return n
 16 }

```