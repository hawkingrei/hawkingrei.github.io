---
title:    "优雅の使用sync.WaitGroup"
date:     "2017-08-26 14:00:00"
tags:     ["Golang"]
---

## Background

自从上次参加2017 GopherChina被安利了NSQ后，阅读了[NSQ](https://github.com/nsqio/nsq)的源代码,从中学到了不少代码技巧。于是乎，我就把这些代码技巧运用到了[veda](https://github.com/hawkingrei/veda)上，提高了代码质量。

## WaitGroup介绍

WaitGroup用来等待一群goroutine结束，主Goroutine调用Add来设置有多少goroutines需要去等待。然后各个goroutines开始运行，当结束时，调用Done。同时，可以使用Wait来堵塞程序，直到全部Goroutines已经结束。下面是一个WaitGroup的小例子。

```golang
var wg sync.WaitGroup
var urls = []string{
        "http://www.golang.org/",
        "http://www.google.com/",
        "http://www.somestupidname.com/",
}
for _, url := range urls {
        // Increment the WaitGroup counter.
        wg.Add(1)
        // Launch a goroutine to fetch the URL.
        go func(url string) {
                // Decrement the counter when the goroutine completes.
                defer wg.Done()
                // Fetch the URL.
                http.Get(url)
        }(url)
}
// Wait for all HTTP fetches to complete.
wg.Wait()
``` 

## 好写法

一个子长度最佳长度在50~150行。而在使用中，按照上面官方的例子中的写法，开发者需要自己在启动前加上Add,结束后加上Done，无形中增加代码行数。其次在启动子goroutines时，如果启动的goroutines逻辑比较复杂，可以单独写一个子函数，提高代码的可读性。

在NSQ中有一个[很好的写法](https://github.com/nsqio/nsq/blob/master/internal/util/wait_group_wrapper.go)，首先要实现一个代理类，把WaitGroup的行为封装进去:

```go
package util

import (
	"sync"
)

type WaitGroupWrapper struct {
	sync.WaitGroup
}

func (w *WaitGroupWrapper) Wrap(cb func()) {
	w.Add(1)
	go func() {
		cb()
		w.Done()
	}()
}
```

使用的时候([完整代码](https://github.com/nsqio/nsq/blob/master/nsqd/nsqd.go))：


```golang
type NSQD struct {
	waitGroup            util.WaitGroupWrapper
}

func New(opts *Options) *NSQD {
	n := &NSQD{
		startTime:            time.Now(),
		topicMap:             make(map[string]*Topic),
		exitChan:             make(chan int),
		notifyChan:           make(chan interface{}),
		optsNotificationChan: make(chan struct{}, 1),
		dl:                   dirlock.New(dataPath),
	}
}

func (n *NSQD) Main() {
	n.waitGroup.Wrap(func() {
		protocol.TCPServer(n.tcpListener, tcpServer,	n.logf)
	})
	n.waitGroup.Wrap(func() { n.queueScanLoop() })
	n.waitGroup.Wrap(func() { n.lookupLoop() })
}

func (n *NSQD) Exit() {
	n.waitGroup.Wait()
}
```

于是乎，golang的WaitGroup现在只要Wrap和Wait就能完成，是不是干净简单很多了呢？


