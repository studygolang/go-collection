# freecache

## [freecache][1]

### 一句话描述

Go缓存库，具有零GC开销和高并发性能

### 简介

#### freecache是什么？

#### 为什么选择freecache？

* 支持存储大量数据条目
* **零 GC**
* 协程安全访问
* 过期时间支持
* **接近LRU**的淘汰算法
* 严格的内存使用
* **迭代器支持**

### Example

```golang
package main

import (
	"fmt"
	"runtime/debug"

	"github.com/coocood/freecache"
)

func main() {
	// 缓存大小，100M
	cacheSize := 100 * 1024 * 1024
	cache := freecache.NewCache(cacheSize)
	debug.SetGCPercent(20)
	key := []byte("abc")
	val := []byte("def")
	expire := 60 // expire in 60 seconds
	// 设置KEY
	cache.Set(key, val, expire)
	got, err := cache.Get(key)
	if err != nil {
		fmt.Println(err)
	} else {
		fmt.Printf("%s\n", got)
	}
	fmt.Println("entry count ", cache.EntryCount())
	affected := cache.Del(key)
	fmt.Println("deleted key ", affected)
	fmt.Println("entry count ", cache.EntryCount())
}
```


### 源码分析

源码分析主要针对核心的存储结构

#### 核心的存储结构

```golang

```

### 思考

### Doc

http://godoc.org/github.com/coocood/freecache

### 比较

#### 相似的库

* https://github.com/golang/groupcache
* https://github.com/allegro/bigcache
* https://github.com/coocood/freecache


[1]: https://github.com/coocood/freecache
