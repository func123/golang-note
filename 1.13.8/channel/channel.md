# channel

​	在并发编程中，多个线程通信的方式一般都是共享变量，而为了正确访问共享变量需要做一些控制（同步原语）。在Go语言中，不仅提供了同步原语这样基础的同步机制，而且还提供了一种不同的并发处理方式——channel。在这种机制中，Go将变量通过channel传递，gorountine通过channel间接获得变量，更关键的是，在任意一个时间点，只能有一个gorountine能通过channel获得变量，在这种情况下gorountine之间不需要共享变量，因此条件竞争从设计上就被杜绝了。

​	Go官方鼓励使用channel去做并发编程，并把它的设计想法简化成了一句话:

> Do not communicate by sharing memory; instead, share memory by communicating.

​	“不要通过共享内存来通信，而要通过通信来实现内存共享”。个人理解：并发编程中不要通过共享变量去做线程间的通信，而是让线程跟channel通信，通过channel传递和接收变量，间接实现线程间的通信。

## CSP

​	channel的设计来源 于Hoare 的通信顺序处理（Communicating sequential processes，CSP）。















1. channel是什么
2. 有哪些操作、用法 resulting value
3. 怎么做的
4. 具体的源码实现

# 数据结构

```go
type hchan struct {
	qcount   uint           // 数据个数
	dataqsiz uint           // 环形队列的长度
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // 发送index
	recvx    uint   // 接收index
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


type waitq struct {
	first *sudog
	last  *sudog
}
```

# API

## 创建

​	基本上是初始化channel，分配内存等

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
	}
	return c
}
```

