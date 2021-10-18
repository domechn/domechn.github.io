---
title: "[Golang] http.Request复用"
subtitle: golang_http_request
date: 2019-02-21 14:49:45
tags:
  - golang
  - http
---

### 复用针对除了 Get 以外的请求

```go
package main

import (
 "net/http"
 "strings"
 )

func main(){
     reader := strings.NewReader("hello")
     req,_ := http.NewRequest("POST","http://www.abc.com",reader)
     client := http.Client{}
     client.Do(req) // 第一次会请求成功
     client.Do(req) // 请求失败
}
```

第二次请求会出错

`http: ContentLength=5 with Body length 0`

原因是第一次请求后 req.Body 已经读取到结束位置，所以第二次请求时无法再读取 body，
解决方法：重新定义一个 ReadCloser 的实现类替换 req.Body

```go
package reader

import (
 "io"
 "net/http"
 "strings"
 "sync/atomic"
)

type Repeat struct{
 reader io.ReaderAt
 offset int64
}

// Read 重写读方法，使每次读取request.Body时能从指定位置读取
func (p *Repeat) Read(val []byte) (n int, err error) {
 n, err = p.reader.ReadAt(val, p.offset)
 atomic.AddInt64(&p.offset, int64(n))
 return
}

// Reset 重置偏移量
func (p *Repeat) Reset(){
        atomic.StoreInt64(&p.offset,0)
}

func (p *Repeat) Close() error {
    // 因为req.Body实现了readcloser接口，所以要实现close方法
    // 但是repeat中的值有可能是只读的，所以这里只是尝试关闭一下。
 if p.reader != nil {
      if rc, ok := p.reader.(io.Closer); ok {
       return rc.Close()
      }
     }
 return nil
}

func doPost()  {
    client := http.Client{}
    reader := strings.NewReader("hello")
    req , _ := http.NewRequest("POST","http://www.abc.com",reader)
    req.Body = &Repeat{reader:reader,offset:0}
    client.Do(req)
    // 将偏移量重置为0
    req.Body.(*Repeat).Reset()
    client.Do(req)
}
```

这样就不会报错了，因为也重写了 Close()方法，所以同时也解决了 request 重用时，req.Body 自动关闭的问题。
