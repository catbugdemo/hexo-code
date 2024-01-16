---
title: 学习Go并发模型
date: 2021-08-06 10:30:01
tags:
---

## 前言

- 简化并发开发
- 以下代码内容均来自[大彬大佬](https://github.com/Shitaibin/golang_step_by_step)
<!--more-->
## 1.简单例子

- 将数组内的数据转变为他们的平方
- 分解以上过程为三个步骤
  - 生产信息 `producer()`，遍历切片
  - 处理信息 `square()`，计算平方
  - 消费信息 `main()`，消费

### 1.生产信息

```
COPYfunc producer(nums ...int) <-chan int {
	// 创建带缓冲通道
	out := make(chan int,10)
	// 通过协程将数据存储到通道中
	go func(){
		defer close(out) //最后关闭通道
		for _,num := range nums {
			out <- num
		}
	}()
	return out
}
```

### 2.处理信息

```
COPYfunc square(inCh <-chan int) <-chan int {
	out := make(chan int,10)
	go func(){
		defer cloes(out)
		for n := range inCh {
			out <- n*n
		}
	}()
	return out
}
```

### 3.消费信息

```
COPYfunc main() {
	// 先将数据拆分放入通道
	in := producer(1,2,3,4)
	// 处理数据
	ch := square(in)
	// 消费数据
	for ret := range ch {
	fmt.Printf("%3d",ret)
	}
}
```

## 扇形模型优化 FAN-IN 与 FAN-OUT

- FAN-OUT : 多个 goruntine 从同一个通道读取数据，直到该通道关闭
- FAN-IN ：1个 goruntine 从多个通道读取数据，直到该通道关闭

### 1. FAN-OUT 和 FAN-IN 实践

#### 1.生产者`producer()` 和 消息处理`square()`不变

```
COPYfunc producer(nums ...int) <-chan int {
	// 创建带缓冲通道
	out := make(chan int,10)
	// 通过协程将数据存储到通道中
	go func(){
		defer close(out) //最后关闭通道
		for _,num := range nums {
			out <- num
		}
	}()
	return out
}

func square(inCh <-chan int) <-chan int {
	out := make(chan int,10)
	go func(){
		defer cloes(out)
		for n := range inCh {
			out <- n*n
		}
	}()
	return out
}
```

#### 2. 新增`merge()` 用来多个`square()` 操作最后回归到一个通道消费读取— FAN-IN

```
COPYfunc merge(cs ...<-chan int) <-chan int {
	out := make(chan int,10)
	
	// 创建计时器
	var wg sync.WaitGroup
	// 将所有数据回归到一个通道中
	// 该通道为可操作通道
	collect := func (in chan int){
		defer wg.Done()
		for n := range in {
			out <- n
		}
	}
	
	wg.Add(len(cs))
	// FAN - IN
	for _,c := range cs {
		go collect(c)
	}
	
	// 错误方式：直接等待是bug，死锁，因为merge写了out，main却没有读
	// wg.Wait()
	 // close(out)
	 
	go func(){
		wg.Wait()
		close(out)
	}()
	
	return out
}
```

#### 3.修改`main()`,启动3个`square()`，一个生产者`producer()`被多个`square()`读取 — FAN-OUT

```
COPYfunc main() {
	in := producer(1,2,3,4)
	
	// FAN-OUT  这个时候开启了协程
	c1 := square(in)
	c2 := square(in)
	c3 := square(in)
	
	// consumer
	for ret := merge(c1,c2,c3) {
		fmt.Printf("%3d",ret)
	}
}
```

## 3.优化 FAN 模式

- 不同的场景优化不同，要依据具体的情况，解决程序的瓶颈
- 但总的来说 不推荐用无缓冲通道，推荐用有缓冲通道

## 结语

- 这是一篇学习博客，推荐去看 [原文章](https://github.com/Shitaibin/golang_step_by_step)
- 谢谢能看到最后
