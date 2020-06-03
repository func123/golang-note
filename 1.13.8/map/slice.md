# slice

​	切片（slice）被抽象成数组的一个片段，在这个基础上，多个切片可以共享一个底层数组，所以一个切片修改了底层数组的数据，其他切片对应的数据也将改变。另外切片之间是不支持比较操作（==）的，但是切片可以跟nil比较。

# 数据结构

```go
type slice struct {
	array unsafe.Pointer	// 底层数组的指针
	len   int				// 长度
	cap   int				// 容量
}
```

# API

## 创建

```go
// 第一种创建：
var a = new([]int)
	
func newobject(typ *_type) unsafe.Pointer {
    return mallocgc(typ.size, typ, true)
}

// 第二种创建：
var a = make([]int, 10, 20)
	
func makeslice(et *_type, len, cap int) unsafe.Pointer {
    // 计算申请的内存大小 以及 是否溢出
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
    // maxAlloc 是最大可申请的内存大小
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}
	// 申请内存
	return mallocgc(mem, et, true)
}
```



## 扩容

1. 检查参数合法
2. 根据扩容策略计算新切片的最终容量
   - 新申请容量（cap）大于2倍的旧容量（old.cap），最终容量（newcap）= 新申请的容量（cap）
   - 旧切片的长度小于1024，最终容量（newcap） = 2倍的旧容量（old.cap）
   - 设置 newcap =  old.cap，循环地将 newcap  变成原来的1.25倍，直到它大于等于新申请容量（cap）
   - newcap   的计算溢出了，最终容量（newcap） =  新申请容量（cap）
3. 计算新切片内存大小，进行内存对齐，申请内存**（由于会进行内存对齐，因此新切片的容量可能会比（2）中计算的最终容量大）**
4. 将旧切片的值复制到新切片中,并且其长度跟旧切片相等。

```go
// growslice 在执行append的时候对slice进行扩容.
// 该方法接收slice类型、旧slice和最小容量cap，并且会返回一个容量为cap的slice,同时也会将旧slice的数据复制过去
// 新slice 的len 跟 旧slice 的len 一样, 不等于参数中的cap
// This is for codegen convenience. The old slice's length is used immediately
// to calculate where to write new values during an append.
// TODO: When the old backend is gone, reconsider this decision.
// The SSA backend might prefer the new length or to return only ptr/cap and save stack space.
func growslice(et *_type, old slice, cap int) slice {
	······
	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}
	// 扩容的策略：优先保证达到cap的要求
	newcap := old.cap
	doublecap := newcap + newcap
    // cap > 2 * old.cap； newcap = cap
	if cap > doublecap {
		newcap = cap
	} else {
        // cap < 2 * old.cap && old.len < 1024 ; newcap =  2 * old.cap
		if old.len < 1024 {
			newcap = doublecap
		} else {
			// 用 0 < newcap 去检测溢出并且防止陷入一个死循环
            // old.cap < cap < 2 * old.cap && old.len > 1024 (第一次for)
            // cap < newcap = 1.25 * old.cap
            for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// 当 newcap 的计算溢出时， newcap = cap
			if newcap <= 0 {
				newcap = cap
			}
		}
	}	
	
    // 计算新切片的长度和容量
	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// 根据类型的字节大小，分别进行内存对齐的策略处理.
	// 对于 1 字节， 不需要任何处理.
	// 对于 sys.PtrSize 字节, 编译器会用乘除操作计算一个常量去移位
	// 对于 2 的幂 个字节, 用变量去移位.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```



## 复制

```go
func slicecopy(to, fm slice, width uintptr) int {
	if fm.len == 0 || to.len == 0 {
		return 0
	}

	n := fm.len
	if to.len < n {
		n = to.len
	}

	if width == 0 {
		return n
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicecopy)
		racewriterangepc(to.array, uintptr(n*int(width)), callerpc, pc)
		racereadrangepc(fm.array, uintptr(n*int(width)), callerpc, pc)
	}
	if msanenabled {
		msanwrite(to.array, uintptr(n*int(width)))
		msanread(fm.array, uintptr(n*int(width)))
	}

	size := uintptr(n) * width
	if size == 1 { // common case worth about 2x to do here
		// TODO: is this still worth it with new memmove impl?
		*(*byte)(to.array) = *(*byte)(fm.array) // known to be a byte pointer
	} else {
		memmove(to.array, fm.array, size)
	}
	return n
}



func slicestringcopy(to []byte, fm string) int {
	if len(fm) == 0 || len(to) == 0 {
		return 0
	}

	n := len(fm)
	if len(to) < n {
		n = len(to)
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicestringcopy)
		racewriterangepc(unsafe.Pointer(&to[0]), uintptr(n), callerpc, pc)
	}
	if msanenabled {
		msanwrite(unsafe.Pointer(&to[0]), uintptr(n))
	}

	memmove(unsafe.Pointer(&to[0]), stringStructOf(&fm).str, uintptr(n))
	return n
}

```



# 空切片  和 nil 切片

空切片和 nil 切片的区别在于，所有空切片底层的数组指针都指向的一个常量的内存地址，但是它没有分配任何内存空间，即底层元素包含0个元素。

![](..\..\images\slice_1.png)

不管是 nil 切片还是空切片，对其调用内置函数 append，len 和 cap 的效果都是一样的。但是以下情景中会有区别：

1. 比较

   ```go
   package main
   
   import "fmt"
   
   func main() {
   	var s1 []int
   	var s2 = []int{}
   
   	fmt.Println(s1 == nil)
   	fmt.Println(s2 == nil)
   
   	fmt.Printf("%#v\n", s1)
   	fmt.Printf("%#v\n", s2)
   }
   
   -------
   true
   false
   []int(nil)
   []int{}
   ```

2. 结构体赋值

   ```go
   type Something struct {
   	values []int
   }
   
   var s1 = Something{}
   var s2 = Something{[]int{}}
   fmt.Println(s1.values == nil)
   fmt.Println(s2.values == nil)
   
   --------
   true
   false
   ```

3. 序列化

   ```go
   type Something struct {
   	Values []int
   }
   
   var s1 = Something{}
   var s2 = Something{[]int{}}
   bs1, _ := json.Marshal(s1)
   bs2, _ := json.Marshal(s2)
   fmt.Println(string(bs1))
   fmt.Println(string(bs2))
   
   ---------
   {"Values":null}
   {"Values":[]}
   ```

   

# Q&A

1. 通过遍历切片时，修改相应的遍历值，会不会修改底层的数组？

   ​	不会修改底层的数组，因为遍历值“i"只是底层数组上数据的值拷贝，不能改变切片中元素的值

   ```go
   package main
   
   func main() {
   	s := []int{1, 1, 1}
   	f(s)
   	fmt.Println(s)
   }
   
   func f(s []int) {
   	for _, i := range s {
   		i++
   	}	
   }
   ```

2. 将切片作为函数参数，会不会影响到函数外部的原切片？

   ​	具体情况具体分析，如果函数内部修改的切片跟原切片共享一个底层数组且两个切片操作的数据范围，在底层数组上是“重叠的”，那么就有影响。如果函数内部操作的切片经过扩容，跟原切片已经不是共享数据底层了，那么就不会影响到原切片了。

3. 切片是怎么扩容的？

   ​	切片通过`append`操作增加新元素，此时当切片容量满了就会触发扩容处理：

   1. 计算扩容后所需的容量，同时在计算中会根据策略（具体看源码）预留一部分的容量来防止频繁的进行扩容导致的开销
   2. 将(1)中的容量进行内存对齐的计算处理后，得到最终的容量大小
   3. 申请一块容量大小 * 类型字节的内存
   4. 把原切片的数据复制到新切片中

4. 数组和切片有何异同？

   1. 切片的数据底层是数组，切片抽象成数组的一个片段，两者都可以通过下标访问元素
   2. 数组是一块连续的内存，而切片是一个结构体，封装了底层数组和自身长度容量的信息
   3. 数组长度是固定的，而切片长度是动态的，非常灵活，但是切片本身会维护长度和容量，如果访问切片超过了其中的长度，也会发生数组越界的错误

# 参考资料

1. [深度解密Go语言之Slice](https://zhuanlan.zhihu.com/p/61121325)
2. [深度解析 Go 语言中「切片」的三种特殊状态](https://juejin.im/post/5bea58df6fb9a049f153bca8)
3. [深入解析 Go 中 Slice 底层实现](https://halfrost.com/go_slice/)
4. [Go-Questions](https://github.com/changkun/Go-Questions/tree/master/%E6%95%B0%E7%BB%84%E5%92%8C%E5%88%87%E7%89%87)



