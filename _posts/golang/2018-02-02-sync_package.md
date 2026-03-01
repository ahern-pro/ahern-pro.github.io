---
title: sync包相关
date: 2018-02-02 00:00:00 +0800
categories: [language, golang]
tags: [golang]
author: ahern
---

## sync.Once - 函数只执行一下

### demo

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var once sync.Once
    onceBody := func() {
        fmt.Println("Only once")
    }
    done := make(chan bool)
    for i := 0; i < 10; i++ {
        go func() {
            once.Do(onceBody)
            done <- true
        }()
    }
    for i := 0; i < 10; i++ {
        <-done
    }
}

# 打印结果
Only once
```

## sync.REMutex - 读写锁

### 概述

- 写锁只能被一个goroutine占用，读锁可以同时被多个goroutine同时获取
- 适用场景：适用用于读多写少的场景

### Api

- Lock( )                // 加写锁
- RLock( )             // 加读锁

## sync.Mutex - 互斥锁

### 概述

- 一个goroutine或得Mutex后，其他的goroutine只能等到这个goroutine释放Mutex
- 已经锁定的 Mutex 并不与特定的 goroutine 相关联，这样可以利用一个 goroutine 对其加锁，再利用其他 goroutine 对其解锁
- 使用场景：适用于一个读一个写的场景

### Api

- Lock( )
- Unlock( )

### demo

```go
package main

import (
    "time"
    "fmt"
    "sync"
)

func main() {
    var mutex sync.Mutex
    fmt.Println("Lock the lock")
    mutex.Lock()
    fmt.Println("The lock is locked")
    channels := make([]chan int, 4)
    for i := 0; i < 4; i++ {
        channels[i] = make(chan int)
        go func(i int, c chan int) {
            fmt.Println("Not lock: ", i)
            mutex.Lock()
            fmt.Println("Locked: ", i)
            time.Sleep(time.Second)
            fmt.Println("Unlock the lock: ", i)
            mutex.Unlock()
            c <- i
        }(i, channels[i])
    }
    time.Sleep(time.Second)
    fmt.Println("Unlock the lock")
    mutex.Unlock()
    time.Sleep(time.Second)

    for _, c := range channels {
        <-c
    }
}
```

## sync.Pool - 临时对象池

### 概述

- 维护一个本地对象池，而不需要频繁的创建对象和gc
- 适用场景：适用于无状态的对象复用：fmt包，不适用于有状态的对象，如：socket、数据库连接池
- 创建Pool需要实现一个New方法，当获取不到临时对象时，调用New方法创建

### 原理


### Api

- Get( )                 // 获取一个临时对象
- Put( )                  // 将临时对象放回pool中

### demo

```go
package main

import (
    "bytes"
    "io"
    "os"
    "sync"
    "time"
)

var bufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func timeNow() time.Time {
    return time.Unix(1136214245, 0)
}

func Log(w io.Writer, key, val string) {
    // 获取临时对象，没有的话会自动创建
    b := bufPool.Get().(*bytes.Buffer)
    b.Reset()
    b.WriteString(timeNow().UTC().Format(time.RFC3339))
    b.WriteByte(' ')
    b.WriteString(key)
    b.WriteByte('=')
    b.WriteString(val)
    w.Write(b.Bytes())
    // 将临时对象放回到 Pool 中
    bufPool.Put(b)
}

func main() {
    Log(os.Stdout, "path", "/search?q=flowers")
}

打印结果：
2006-01-02T15:04:05Z path=/search?q=flowers
```

## sync.Cond - 条件变量

### 概述

- sync.Cond实现了goroutine状态变化的通信机制，如：goroutine A的执行通过Wait( )等待goroutine B的通知，goroutine B维护一个通知列表，调用Signal( )或Broadcast( )通知goroutine A恢复执行。
- sync.Cond总是与锁一起使用，并在Wait( )之前就上锁

### Api

- sync.NewCond( )		// 创建cond对象
- cond.L.Lock( )	         // 上锁
- Wait( )                         // 协程阻塞
- Signal( )                      // 唤醒列表中的一个协程
- Broadcast( )                // 唤醒所有协程

### demo

```go
package main

import (
    "bytes"
    "fmt"
    "io"
    "sync"
    "time"
)

type MyDataBucket struct {
    br     *bytes.Buffer
    gmutex *sync.RWMutex
    rcond  *sync.Cond //读操作需要用到的条件变量
}

func NewDataBucket() *MyDataBucket {
    buf := make([]byte, 0)
    db := &MyDataBucket{
        br:     bytes.NewBuffer(buf),
        gmutex: new(sync.RWMutex),
    }
    db.rcond = sync.NewCond(db.gmutex.RLocker())
    return db
}

func (db *MyDataBucket) Read(i int) {
    db.gmutex.RLock()
    defer db.gmutex.RUnlock()
    var data []byte
    var d byte
    var err error
    for {
        //读取一个字节
        if d, err = db.br.ReadByte(); err != nil {
            if err == io.EOF {
                if string(data) != "" {
                    fmt.Printf("reader-%d: %s\n", i, data)
                }
                db.rcond.Wait()
                data = data[:0]
                continue
            }
        }
        data = append(data, d)
    }
}

func (db *MyDataBucket) Put(d []byte) (int, error) {
    db.gmutex.Lock()
    defer db.gmutex.Unlock()
    //写入一个数据块
    n, err := db.br.Write(d)
    db.rcond.Broadcast()
    return n, err
}

func main() {
    db := NewDataBucket()

    go db.Read(1)

    go db.Read(2)

    for i := 0; i < 10; i++ {
        go func(i int) {
            d := fmt.Sprintf("data-%d", i)
            db.Put([]byte(d))
        }(i)
        time.Sleep(100 * time.Millisecond)
    }
}
```

## sync.WaitGroup

### 概述

- 阻塞主线程，直到所有的goroutine执行完成

### Api

- Add( )              // 计时器加n
- Done( )            // 计时器减1
- Wait( )             // 线程阻塞，直到计时器为0

### demo

```go
func main() {
    wg := sync.WaitGroup{}
    wg.Add(100)
    for i := 0; i < 100; i++ {
        go f(i, &wg)
    }
    wg.Wait()
}

// 一定要通过指针传值，不然进程会进入死锁状态
func f(i int, wg *sync.WaitGroup) { 
    fmt.Println(i)
    wg.Done()
}
```
