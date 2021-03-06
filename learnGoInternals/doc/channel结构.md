##channel数据结构
Go语言channel是first-class的，意味着它可以被存储到变量中，可以作为参数传递给函数，也可以作为函数的返回值返回。作为Go语言的核心特征之一，虽然channel看上去很高端，
但是其实channel仅仅就是一个数据结构而已，结构体定义如下：
```go
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```
其中一个核心的部分是存放channel数据的环形队列，由qcount和elemsize分别指定了队列的容量和当前使用量。
dataqsize是队列的大小。elemalg是元素操作的一个Alg结构体，记录下元素的操作，如copy函数，equal函数，hash函数等

另一个重要部分就是recvq和sendq两个链表，一个是因读这个通道而导致阻塞的goroutine，
另一个是因为写这个通道而阻塞的goroutine。如果一个goroutine阻塞于channel了，那么它就被挂在recvq或sendq中。WaitQ是链表的定义，包含一个头结点和一个尾结点：
```go
type waitq struct {
	first *sudog
	last  *sudog
}
```
//in runtime2.go
```go
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g          *g
	selectdone *uint32 // CAS to 1 to win select race (may point to stack)
	next       *sudog
	prev       *sudog
	elem       unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
}
```
该结构中主要的就是一个g和一个elem。elem用于存储goroutine的数据。读通道时，数据会从Hchan的队列中拷贝到SudoG的elem域。写通道时，数据则是由SudoG的elem域拷贝到Hchan的队列中

Hchan结构如下图所示
![](images/7.1.channel.png?raw=true)

###读写操作
先看写channel的操作，基本的写channel操作，在底层运行时库中对应的是一个runtime.chansend函数
```go
c <- v
```
在运行时库中会执行：
```go 1.9 chan.go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {}
```
其中c就是channel，ep是取变量v的地址。这里的传值约定是调用者负责分配好ep的空间，仅需要简单的取变量地址就够了
这个函数首先会区分是同步还是异步。同步是指chan是不带缓冲区的，因此可能写阻塞，而异步是指chan带缓冲区，只有缓冲区满才阻塞
在同步的情况下，由于channel本身是不带数据缓存的，这时首先会查看Hchan结构体中的recvq链表时否为空，即是否有因为读该管道而阻塞的goroutine。如果有则可以正常写channel，否则操作会阻塞
recvq不为空的情况下，将一个SudoG结构体出队列，将传给通道的数据(函数参数ep)拷贝到SudoG结构体中的elem域，并将SudoG中的g放到就绪队列中，状态置为ready，然后函数返回。
如果recvq为空，否则要将当前goroutine阻塞。此时将一个SudoG结构体，挂到通道的sendq链表中，这个SudoG中的elem域是参数eq，SudoG中的g是当前的goroutine。当前goroutine会被设置为waiting状态并挂到等待队列中
在异步的情况，如果缓冲区满了，也是要将当前goroutine和数据一起作为SudoG结构体挂在sendq队列中，表示因写channel而阻塞。否则也是先看有没有recvq链表是否为空，有就唤醒
跟同步不同的是在channel缓冲区不满的情况，这里不会阻塞写者，而是将数据放到channel的缓冲区中，调用者返回
读channel的操作也是类似的，对应的函数是runtime.chansend。一个是收一个是发，基本的过程都是差不多的
需要注意的是几种特殊情况下的通道操作--空通道和关闭的通道
空通道是指将一个channel赋值为nil，或者定义后不调用make进行初始化。按照Go语言的语言规范，读写空通道是永远阻塞的。其实在函数runtime.chansend和runtime.chanrecv开头就有判断这类情况，如果发现参数c是空的，则直接将当前的goroutine放到等待队列，状态设置为waiting
读一个关闭的通道，永远不会阻塞，会返回一个通道数据类型的零值。这个实现也很简单，将零值复制到调用函数的参数ep中。写一个关闭的通道，则会panic。关闭一个空通道，也会导致panic

###select实现
select-case中的chan操作编译成了if-else。比如：
```go
select {
case v = <-c:
        ...foo
default:
        ...bar
}
```
会被编译为:
```go
if selectnbrecv(&v, c) {
        ...foo
} else {
        ...bar
}
```
类似地:
```go
select {
case v, ok = <-c:
	... foo
default:
	... bar
}
```
会被编译为:
```go
if c != nil && selectnbrecv2(&v, &ok, c) {
	... foo
} else {
	... bar
}
```
接下来就是看一下selectnbrecv(chan.go)相关的函数了。其实没有任何特殊的魔法，这些函数只是简单地调用runtime.chanrecv函数，只不过设置了一个参数，告诉当runtime.chanrecv函数，当不能完成操作时不要阻塞，而是返回失败。
也就是说，所有的select操作其实都仅仅是被换成了if-else判断，底层调用的不阻塞的通道操作函数
在Go的语言规范中，__select中的case的执行顺序是随机的__，而不像switch中的case那样一条一条的顺序执行。那么，如何实现随机呢？
select和case关键字使用了下面的结构体：
1.9 select.go
```go
type scase struct {
	elem        unsafe.Pointer // data element
	c           *hchan         // chan
	pc          uintptr        // return pc (for race detector / msan)
	kind        uint16
	receivedp   *bool // pointer to received bool, if any
	releasetime int64
}
```
```go
type hselect struct {
	tcase     uint16   // total count of scase[]
	ncase     uint16   // currently filled scase[]
	pollorder *uint16  // case poll order
	lockorder *uint16  // channel lock order
	scase     [1]scase // one per case (in order of appearance)
}
```
__每个select都对应一个Select结构体。在Select数据结构中有个Scase数组，记录下了每一个case，而Scase中包含了Hchan。然后pollorder数组将元素随机排列，这样就可以将Scase乱序了__


