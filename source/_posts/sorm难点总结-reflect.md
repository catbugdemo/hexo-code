---
title: sorm难点总结-reflect
date: 2022-02-08 14:54:13
tags:
---

## 假如 interface 进来了，我想传给下一步，但其本身还能赋值
<!--more-->
```
// value 为指针
func Scan(values interface{}) error{
    // 返回持有values持有的指针指向的值的Value
    value := reflect.Indirect(reflect.ValueOf(values))
    // 将反射的地址传给下一步
    s.QueryRow().Scan(value.Addr().Interface())
}
```

## 想要通过反射创建一个切片

```
// values 本身为一个结构体
func CreateSlice(values interface{}){
    // 返回持有values持有的指针指向的值的Value
    value := reflect.Indirect(reflect.ValueOf(values))
    // 根据value中具有的类型创建一个新的切片的指针
    valueSlice :=reflect.New(reflect.SliceOf(value.Type())).Elem()
    // 将反射的地址传给下一步
    s.QueryRow().Scan(valueSlice.Addr().Interface())
}
```

## interface 赋值

```
COPYfunc Get(value interface{}){
    // 返回持有values持有的指针指向的值的Value
    value := reflect.Indirect(reflect.ValueOf(values))
    // 赋值操作
    value.SetInt(0)  // ... 可以赋予任何种类参考 golang 标准库
}
```

## interface 为切片时赋值

```
COPYfunc GetSlice(value interface{}){
    // 返回持有values持有的指针指向的值的Value
    value := reflect.Indirect(reflect.ValueOf(values))
    for i := 0; i < value.NumField();i++{
        value.Index(i).SetInt(0) // 切片赋值
    }
}
```

## 参考

https://github.com/catbugdemo/sorm
