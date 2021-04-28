# go-cache 

## [go-cache ](https://github.com/patrickmn/go-cache)

### 一句话描述

基于内存的 K/V 存储/缓存 : (类似于Memcached)，适用于单机应用程序

### 简介

#### go-cache是什么？

基于内存的 K/V 存储/缓存 : (类似于Memcached)，适用于单机应用程序 ，支持删除，过期，默认Cache共享锁，

大量key的情况下会造成锁竞争严重


#### 为什么选择go-cache？

可以存储任何对象（在给定的持续时间内或永久存储），并且可以由多个goroutine安全地使用缓存。

### Example 

```golang
package main

import (
	"fmt"
	"time"

	"github.com/patrickmn/go-cache"
)

type MyStruct struct {

	Name string
}

func main() {
	// 设置超时时间和清理时间
	c := cache.New(5*time.Minute, 10*time.Minute)

	// 设置缓存值并带上过期时间
	c.Set("foo", "bar", cache.DefaultExpiration)


	// 设置没有过期时间的KEY，这个KEY不会被自动清除，想清除使用：c.Delete("baz")
	c.Set("baz", 42, cache.NoExpiration)


	var foo interface{}
	var found bool
	// 获取值
	foo, found = c.Get("foo")
	if found {
		fmt.Println(foo)
	}

	var foos string
	// 获取值， 并断言
	if x, found := c.Get("foo"); found {
		foos = x.(string)
		fmt.Println(foos)
	}
	// 对结构体指针进行操作
	var my *MyStruct
	c.Set("foo", &MyStruct{Name: "NameName"}, cache.DefaultExpiration)
	if x, found := c.Get("foo"); found {
		my = x.(*MyStruct)
		// ...
	}
	fmt.Println(my)
}

```


### 源码分析

源码分析主要针对核心的存储结构、CRUD、定时清理逻辑进行分析。包含整体的逻辑架构

#### 核心的存储结构

```golang
package cache

// Item 每一个具体缓存值
type Item struct {
	Object     interface{}
	Expiration int64 // 过期时间:设置时间+缓存时长
}

// Cache 整体缓存
type Cache struct {
	*cache
}

// cache 整体缓存
type cache struct {
	defaultExpiration time.Duration // 默认超时时间
	items             map[string]Item // KV对
	mu                sync.RWMutex // 读写锁，在操作（增加，删除）缓存时使用
	onEvicted         func(string, interface{})
	janitor           *janitor // 定时清空缓存的结构
}

// janitor  定时清空缓存的结构
type janitor struct {
	Interval time.Duration // 多长时间扫描一次缓存
	stop     chan bool // 是否需要停止
}

```


### Doc

http://godoc.org/github.com/patrickmn/go-cache

### QA

大量key的情况下会造成锁竞争严重


### 比较

> 备注： 可以到https://go.libhunt.com/这个网站进行库对比或者链接到其他博客网站


### 相似的库


