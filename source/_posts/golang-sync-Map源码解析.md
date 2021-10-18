---
title: golang sync.Map源码解析
subtitle: golang_sync_map
date: 2019-03-01 14:04:44
tags:
- golang
---

golang的内建类型map是非线程安全的，当我们使用并发操作去写map时会引发panic

```go
package main

import (
 "fmt"
 "sync"
)

func main() {
 var m = make(map[string]string,100)

 var wait  sync.WaitGroup

 for i := 0 ;i < 100 ; i ++ {
  wait.Add(1)
  go func() {
   m[fmt.Sprintf("%d",i)] = "test"
   wait.Done()
  }()
 }

 wait.Wait()
}
```

```plaintext
fatal error: concurrent map writes
fatal error: concurrent map writes

goroutine 11 [running]:
runtime.throw(0x10c6370, 0x15)
 /usr/local/go/src/runtime/panic.go:608 +0x72 fp=0xc00002cf08 sp=0xc00002ced8 pc=0x1026e52
runtime.mapassign_faststr(0x10a9860, 0xc00006e120, 0xc0000a0010, 0x2, 0x1)
 /usr/local/go/src/runtime/map_faststr.go:199 +0x3da fp=0xc00002cf70 sp=0xc00002cf08 pc=0x100fada
main.main.func1(0xc00006e120, 0xc000016078, 0xc000016080)
...
```

因此当我们需要并发修改map时，第一种方法是在每次修改map值时使用互斥锁，第二种则是直接使用golang内置的线程安全的map，也就是sync.Map。

本文则从sync.Map的源码层面介绍，sync.Map是如何实现线程安全的。

首先我们看下sync.Map的数据结构

```go
// Map 类似于map[interface{}]interface{}，但是对于多个goroutine并发使用是安全的，不需要额外的lock
// 
// 该类型是专门为并发操作提供的，你应该使用该类型来替代需要对原生map加锁的场景，
// 以此获得更好地安全性，并使得操作更加简便
//
// Map针对了两个常见的场景进行了优化（1）对一个key只写入以此，但是读很多次。（2）在并发场景下，goroutine不会
// 同时去操作一个key（也就是不会发生竞态）。
// 在这两种情况下使用Map会比使用RWMutex加锁的map效率更高
//
// 你可以直接var x sync.Map就能使用，不需要去执行New...,Map不能被copy
type Map struct {
 mu Mutex

    // read中保存了Map中部分内容，这些内容是只读的，所以是线程安全的
    // 其中保存的数据类型是readOnly
 read atomic.Value // readOnly

    // 所有对dirty的操作都是需要加锁的
    // 如果dirty为空，下一次写操作会复制read中没被删除的数据到dirty
 dirty map[interface{}]*entry

    // 当从Map中读取entry时，会先去read中读取，如果read中读不到则到dirty中读取，这是
    // 会将该值+1，当该值到达一定大小后就会将read中所有值更新为dirty中保存的值
 misses int
}
```

从Map的数据结构看，就不难发现Map是如何保证线程安全的

当往Map中添加新的值时，不会往read中插入数据，而是直接将数据保存在dirty中。当需要从Map中读取数据时，
会先从read中读取数据，如果读到直接返回，如果读不到那么就会从dirty中读取，并更新misses的值，当misses值到达一定数值之后，
就会将dirty的值赋给read。(所有对dirty的操作都是加锁的，这就保证了这个类是线程安全的，同时因为read-only的存在，提升了并发读取的效率)

```go
// readOnly is an immutable struct stored atomically in the Map.read field.
type readOnly struct {
 m       map[interface{}]*entry
 amended bool // 如果dirty中存在一些m中没有的key，该值则为true
}
```

```go
// An entry is a slot in the map corresponding to a particular key.
type entry struct {
 // p points to the interface{} value stored for the entry.
 //
    // p有三种情况
    // p == nil: entry已经被删除，且m.dirty为nil
    // expunged: entry已经被删除，但是m.dirty不是nil，并且这个entry不在m.dirty中
    // 其他: entry是个正常值
 p unsafe.Pointer // *interface{}
}
```

## Load

Load方法用于根据Key读取Map中的值

```go
// Load 根据所给的key返回Map中的值，如果不存在返回nil
//
// ok 表示Map中是否包含key
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
    // 直接从readonly中读取，如果读到直接返回，因为是readonly所以不用加锁
 read, _ := m.read.Load().(readOnly)
 e, ok := read.m[key]
    // 如果readonly中没有值，并且dirty中存在read中不存在的值时
 if !ok && read.amended {
  m.mu.Lock()
        // 加锁，双检查
  read, _ = m.read.Load().(readOnly)
  e, ok = read.m[key]
        // 如果read中仍然不存在该key，且dirty中有read中不存在的值
  if !ok && read.amended {
            // 从dirty中检查是否有该key
   e, ok = m.dirty[key]
            // 不管dirty中是否有改key，都将misses+1
            // 当misses到达一定值之后，m.dirty会被提升为read
   m.missLocked()
  }
  m.mu.Unlock()
 }
 if !ok {
  return nil, false
 }
 return e.load()
}
```
