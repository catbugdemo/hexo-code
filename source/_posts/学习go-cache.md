---
title: 学习go-cache
date: 2021-07-29 10:12:27
tags:
---

## go-cache 学习缓存的前置知识

> [go-cache代码地址](https://github.com/patrickmn/go-cache)

### 1.什么是读锁，什么是写锁，sync.Mutex 和 sync.RWMutex 的区别
<!--more-->
> [参考地址](https://www.jianshu.com/p/679041bdaa39)

### 2.go runtime 协程

> [参考地址](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)

### 3. Golang time 和 runtime 包

> [参考地址](https://studygolang.com/pkgdoc)

## 写 go-cache 的顺序

1.确定数据需要存储的内容和其过期时间
2.将cache封装为一个结构体
3.New
4.编写主方法：
添加，修改，删除，查询，过期缓存回收
5.测试
6.验收

## 开始编写

### 1.确定缓存结构体

```
COPY
type Item struct {
	// 能够存储的任何种类的数据
	Object interface{}
	// 源码中过期时间,为什么是 int64 (因为 time.Duration 的代指就是 int64 )
	Expiration int64
}

// 顺便设置常量
const (
	// 设置永久过期时间为 -1
	NoExpiration time.Duration = -1
	// 设置默认过期时间为 0
	DefaultExpiration time.Duration = 0
)
```

### 2.将 `cache` 封装为一个结构体

```
COPY
// 该结构体方便于外部调用
type Cache struct {
	*cache
}

// 2.cache 中存储的内容:
//		默认过期时间，存储的 item, 读写锁, janitor
type cache struct {
	defaultExpiration time.Duration             // 默认过期时间
	items             map[string]Item           // 内容
	mu                sync.RWMutex              // 读写锁
	onEvicted         func(string, interface{}) // 对删除数据二次操作
	janitor           *janitor                  // gc 垃圾回收
}
```

### 3.New

go-cache 中包含 1 个对外的New 和两个内部创建结构体

```
COPY// newCache 该方法真正返回 cache 结构体
// de 设置过期时间 ，m 存储的数据
func newCache(de time.Duration, m map[string]Item) *cache {
	// 如果过期时间是0，创建永久时间
	if de == 0 {
		de = -1
	}
	return &cache{
		defaultExpiration: de,
		items:             m,
	}
}

// 该方法用来运行缓存回收
func newCacheWithJanitor(de, ci time.Duration, m map[string]Item) *Cache {
	c := newCache(de, m)

	C := &Cache{c}

	if ci > 0 {
		// 运行垃圾回收方法，之后会写
		runJanitor(c,ci)
		// 设置垃圾回收,stopJanitor 也是之后写的停止运行缓存回收
		runtime.SetFinalizer(C,stopJanitor)
	}
	return C
}

// New
// defaultExpiration 设置所有缓存的默认过期时间
// cleanInterval 设置缓存清理间隔时间
func New(defaultExpiration, cleanInterval time.Duration) *Cache {
	items := make(map[string]Item)
	return newCacheWithJanitor(defaultExpiration, cleanInterval, items)
}
```

### 4.编写主方法

#### 1.获取

```
COPY
// Get 获取 该数据为,开启读锁是为了保证读取数据的原子性
func (c *cache) Get(k string) (interface{}, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	item, found := c.items[k]
	if !found {
		return nil, false
	}
	// 如果设置了过期时间,判断是否过期
	if item.Expiration > 0 {
		if time.Now().UnixNano() > item.Expiration {
			return nil, false
		}
	}
	return item.Object, true
}


// get 不需要读写锁的原因是因为调用该方法的前置方法已经开启了锁
func (c *cache) get(k string) (interface{}, bool) {
	item, found := c.items[k]
	if !found {
		return nil, false
	}
	if item.Expiration > 0 { //设置了过期时间，但是过期了
		if time.Now().UnixNano() > item.Expiration {
			return nil, false
		}
	}
	return item.Object, true
}
```

#### 2.添加

```
COPY// Set (添加) 该方法因为在外部层面需要保证数据的一致性，防止多个操作
// 该方法伟伦是否有值，都会重新设置
// k - key , x - value , d - 过期时间
func (c cache) Set(k string, x interface{}, d time.Duration) {
	// 最后需要存入的数据内容
	var e int64
	// 如果为 0，则设置为之前设置的默认过期时间
	if d == DefaultExpiration {
		d = c.defaultExpiration
	}
	// 设置过期时间从当前时间开始
	if d > 0 {
		e = time.Now().Add(d).UnixNano()
	}
	// 添加写锁，保证数据的原子性,并写入内容
	c.mu.Lock()
	defer c.mu.Unlock()
	c.items[k] = Item{
		Object:     x, //存储内容
		Expiration: e, //过期时间
	}
}

// set 与 Set 相同，因为作用在 Add 中 已经加了锁，所以不需要加锁
func (c *cache) set(k string, x interface{}, d time.Duration) {
	var e int64
	if d == DefaultExpiration {
		d = c.defaultExpiration
	}
	if d > 0 {
		e = time.Now().Add(d).UnixNano()
	}
	c.items[k] = Item{
		Object:     x,
		Expiration: e,
	}
}

// Add 如果数据存在则不会进行操作
func (c cache) Add(k string, x interface{}, d time.Duration) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	if _, found := c.get(k); found { // 已经存储了数据
		return fmt.Errorf("Item %s already exists", k)
	}
	c.set(k, x, d)
	return nil
}
```

#### 3.修改

```
COPY
// Replace 修改
func (c *cache) Replace(k string, x interface{}, d time.Duration) error {
	c.mu.Lock()
	defer c.mu.Unlock()
	if _, found := c.get(k); !found {
		return fmt.Errorf("Item %s doesn't exist", k)
	}
	// 否则进行设置
	c.set(k, x, d)
	return nil
}
```

#### 4.删除

```
COPY
// Delete 删除缓存, evicted 用来存储驱逐数据
func (c *cache) Delete(k string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	v, evicted := c.delete(k)
	// 是否是需要做二次操作的数据
	if evicted {
		c.onEvicted(k, v)
	}
}

// delete 删除缓存
func (c *cache) delete(k string) (interface{}, bool) {
	if c.onEvicted != nil {
		if v, found := c.items[k]; found {
			delete(c.items, k)
			return v.Object, true
		}
	}
	delete(c.items, k)
	return nil, false
}
```

#### 5.过期缓存回收

```
COPY
type janitor struct {
	Interval time.Duration
	stop     chan bool
}

// Run
// select 作为阻塞进程
func (j *janitor) Run(c *cache) {
	// 设置的每过一段时间返回一个 channel 通道字段
	ticker := time.NewTicker(j.Interval)
	for {
		select {
		// 如果返回的是时间,则运行删除过期缓存
		case <-ticker.C:
			c.DeleteExpired()
		// 如果j中确定停止回收
		case <-j.stop:
			ticker.Stop()
			return
	go j.Run(c)
}

// 对数据进行二次操作
type keyAndValue struct {
	key   string
	value interface{}
}

// DeleteExpired() 删除过期缓存
func (c *cache) DeleteExpired() {
	var evictedItems []keyAndValue
	now := time.Now().UnixNano()
	c.mu.Lock()
	defer c.mu.Unlock()
	for k, v := range c.items {
		if v.Expiration > 0 && now > v.Expiration {
			ov, eveicted := c.delete(k)
			// 将需要进行二次操作的数据存入evictedItems 中
			if eveicted {
				evictedItems = append(evictedItems, keyAndValue{k, ov})
			}
		}
	}
	// 将需要二次操作的数据存储
	for _, v := range evictedItems {
		c.onEvicted(v.key, v.value)
	}
}


// 存储二次操作的数据
func (c *cache) OnEvicted(f func(string, interface{})) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.onEvicted = f
}
```

#### 6.几个常用方法

```
COPY
// Items 获取所有数据
func (c *cache) Items() map[string]Item {
	c.mu.RLock()
	defer c.mu.RUnlock()
	m := make(map[string]Item, len(c.items))
	now := time.Now().UnixNano()
	for k, v := range c.items {
		if v.Expiration > 0 && now > v.Expiration {
			continue
		}
		m[k] =v
	}
	return m
}

// 获取数据总数
func (c cache) ItemCount() int {
	c.mu.RLock()
	defer c.mu.RUnlock()
	return len(c.items)
}

// Flush 删除所有
func (c cache) Flush() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.items = map[string]Item{}
}
```

### 5.单元测试

## 结语

- go-cache 其实常用在二次缓存，同时go-cahce 不需要添加分布式缓存，如果添加了为什么不直接用redis的分布式缓存
- 我们其实并不需要经常重复造轮子，重复造轮子的目的是为了了解成熟框架的运行原理，如果有能力，可以同时进行优化
- 同时也谢谢能看到最后


