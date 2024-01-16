---
title: go语言json技巧
date: 2022-02-07 14:52:04
tags:
---

## 1.基本序列化
<!--more-->
```
type Person struct {
    Name string `json:"nname"`
    Age int	`json:"age"`
    Weight float `json:"weight"`
}

func main(){
    p1 := Person{
        Name: "七米",
        Age: 18,
        Weight: 71.5
    }
    
    buf ,err := json.Marshal(p1)
    if err != nil {
        fmt.Printf("%v",err)
        return 
    }
    fmt.Printf("%s",string(buf))
    
    // 反序列
    var p2 Person 
    if err := json.Unmarshal(b,&p2);err!=nil{
        fmt.Printf("%v",err)
        return
    }
    fmt.Printf("%v",p2)
}
```

## 2.忽略某个字段

```
COPY// 忽略 Weight
type Person struct{
    Name string `json:"name"`
    Weight float `json:"-"` // 指定json序列化/反序列化时忽略此字段
}
```

## 3.忽略空值字段

```
COPY// 当 Weight 为空时，忽略
type Person struct{
    Name string `json:"name"`
    Weight float `json:"weight,omitempty"`
}
```

## 4.自定义MarshalJSON 和 UnmarshalJSON 方法

```
COPYtype Order struct {
	Id int `json:"id"`
	Title string `json:"title"`
	CreatedTime time.Time `json:"created_time"`
}

const layout = "2006-01-02 15:04:05"

func (o *Order) MarshalJSON()([]byte,error) {
	type TempOrder Order // 定义与 Order 字段一致的新类型
	return json.Marshal(struct {
		CreatedTime string `json:"created_time"`
		*TempOrder // 避免直接嵌套 Order 进入死循环
	}{
		CreatedTime: o.CreatedTime.Format(layout),
		TempOrder:(*TempOrder)(o),
	})
}

func (o *Order) UnmarshalJSON(data []byte) error {
	type TempOrder Order 
	ot := struct {
		CreatedTime string `json:"created_time"`
		*TempOrder // 避免直接嵌套 Order 进入死循环
	}{
		TempOrder:(*TempOrder)(o),
	}
	if err := json.Unmarshal(data, &ot);err!=nil{
		return err
	}
	var err error 
	o.CreatedTime, err = time.Parse(layout, ot.CreatedTime)
	if err != nil {
		return err
	}
	return nil
}
```

## 参考

https://www.liwenzhou.com/posts/Go/json_tricks_in_go/
