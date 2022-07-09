---
title: "Go职责链模式,多级缓存"
date: 2022-07-09T16:56:09+08:00
authors: ["金锋"]
categories: ["go", "设计模式"]
draft: false
---

# 职责链模式

职责链模式，顾名思义，一条链，依次按照职责去执行。

打个比方，办理业务，如果窗口可以办理，则窗口办理完成。如果窗口没有权限或者其他原因不能办理，则找来上级办理，上级还不能，则找来上级的上级办理。

在编程上，可以考虑用职责链模式来处理多级缓存。


# 直接上代码

cache.go
```go
package cache

import (
	"golib/tlog"
)

type Cacher interface {
	HandleSet(k string, v []byte, expire int64) error
	HandleGet(k string) ([]byte, bool)
}

type Chain struct {
	Cacher
	ExpireParams float32
	NextCache    *Chain
}

func (c *Chain) SetNext(m *Chain) {
	c.NextCache = m
}

func (c *Chain) Set(k string, v []byte, expire int64) {
	expire = int64(float32(expire) * c.ExpireParams)
	err := c.HandleSet(k, v, expire)
	if err != nil {
		tlog.Errorf("Chain Set Error, %+v, %s, %s, %+v", err, k, string(v), c.Cacher)
	}
	if c.NextCache != nil {
		c.NextCache.Set(k, v, expire)
	}
}

func (c *Chain) Get(k string, expire int64) ([]byte, bool) {
	if c == nil {
		return []byte{}, false
	}
	var (
		data  []byte
		found bool
	)
	if data, found = c.HandleGet(k); found {
		return data, found
	}

	if data, found = c.NextCache.Get(k, expire); found {
		err := c.HandleSet(k, data, expire)
		if err != nil {
			tlog.Errorf("Chain Set Error, %+v, %s, %s, %+v", err, k, string(data), c.Cacher)
		}
	}
	return data, found
}

```


local_cache.go
```go
package cache

import (
	"lib/processcache"
	"math/rand"
	"time"
)

type LocalCache struct {
	CacheMem *processcache.ProcessCache
}

func (c *LocalCache) HandleSet(k string, v []byte, expire int64) error {
	c.CacheMem.Set(k, v, time.Now().Unix()+expire+int64(rand.Intn(int(expire/2))))
	return nil
}

func (c *LocalCache) HandleGet(k string) ([]byte, bool) {
	return c.CacheMem.Get(k)
}

```

redis_cache.go
```go
package cache

import (
	"common/utils"
	"github.com/go-redis/redis"
	"golib/tlog"
	"math/rand"
)

type RedisCache struct {
	RedisClient *utils.RedisClient
}

func (c *RedisCache) HandleSet(k string, v []byte, expire int64) error {
	_, err := c.RedisClient.Set(k, string(v), int(expire)+rand.Intn(int(expire/2)))
	return err
}

func (c *RedisCache) HandleGet(k string) ([]byte, bool) {
	var (
		data string
		err  error
	)
	data, err = c.RedisClient.Get(k)
	if err != nil && err != redis.Nil {
		tlog.Error(err, k)
		return []byte(data), false
	}
	if err == redis.Nil {
		return []byte(data), false
	}
	return []byte(data), true
}

```

使用方式

```go
localCache := cache.LocalCache{CacheMem: cacheMem}
redisCache := cache.RedisCache{RedisClient: redisClient}

cacheChainLocal := cache.Chain{Cacher: &localCache, ExpireParams: 1}
cacheChainRedis := cache.Chain{Cacher: &redisCache, ExpireParams: 1.2}
cacheChainLocal.NextCache = &cacheChainRedis

if data, found := ThisServer.CacheChain.Get(key, expire); found {
    ...
}else{
    ...
}
cacheChainLocal.CacheChain.Set(k, v, expire)

```