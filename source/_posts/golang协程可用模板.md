---
title: golang协程可用模板
date: 2022-01-18 14:47:31
tags:
---

## 前言

- 在我们使用协程的时候，如果一时想不起来，可以使用写好的模板套用
<!--more-->
## sync.WaitGroup

```
COPYfunc FuncName(list []List){
    var wg sync.WaitGroup
    wg.Add(len(list))
    for i,v range list {
        go func(index int,value List){
            defer func() {
                if err:= recover();err!=nil{
                    fmt.Printf("%+v",errors.WithStack(err))
                    return
                }
                wg.Done()
            }()
            
            // do something 
        }(i,v)
    }
}
```

## sync.RWMutex (适用于读多写少情况)

```
COPYtype TmpInfo struct{
    TmpId int 
    rmu sync.RWMutex
    wg sync.WaitGroup
}

func FuncRW(list []List){
    var ti TmpInfo 
    ti.wg.Add(len(list))
    for k,v range list {
        go func(key int,value List){
            // 在使用读写锁的时候，读锁和写锁应该分开使用，如果添加了读锁 RLock() 后继续添加写锁 Lock()，则会出现 循环等待
            // 读锁占用的时候会阻止写，不会阻止读，写锁占用的时候会阻止任何操作
            ti.rmu.RLock()
            if v.Id != ti.TmpId {
                ti.rmu.RUnLock()
                ti.rmu.Lock()
                // do something
                id := 1
                ti.rmu.UnLock()
                ti.TmpId = id 
            }else {
                defer ti.rmu.RunLock()
            }
            // do something
        }(k,v)
    }
}
```

## channel 无缓冲通道 （信号量用法）

```
COPYfunc Ch() {
  ch := make(struct{})
  
  go func(){
    defer func(){
      ch <- struct{}{}
    }()
  }()
  
    go func(){
    defer func(){
      ch <- struct{}{}
    }()
  }()
  
  fo i=0; i< 2;i++{
    <-ch
  }
  close(ch)
}
```

## 参考

https://juejin.cn/post/6844903925364031496
