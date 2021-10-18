---
title: 解决golang使用elastic连接elasticsearch时自动转换连接地址
subtitle: golang_elastic_url
date: 2019-02-28 14:53:27
tags:
- golang
- elasticsearch
---

使用[olivere/*elastic*](https://github.com/olivere/elastic)连接elasticsearch时，发现连接地址明明输入的时候是公网地址，但是连接时会自动转换成内网地址或者docker中的ip地址，导致服务连接不上。

```
// 自动转换成docker中的ip导致无法连接服务
time="2019-02-15T20:04:26+08:00" level=error msg="Head http://172.17.0.2:9200: context deadline exceeded"
```

## 解决方法

```golang
client, _ := elastic.NewClient(
  // ...
  // 将sniff设置为false后，便不会自动转换地址
   elastic.SetSniff(false),
)
```

## 源码解析

```golang
// sniff 会请求http://ip:port/_nodes/http，将其返回的url list作为新的url list。
// 如果snifferEnabled被设置为false，那么则不启动该功能。
func (c *Client) sniff(parentCtx context.Context, timeout time.Duration) error {
 c.mu.RLock()
 if !c.snifferEnabled {
  c.mu.RUnlock()
  return nil
 }

 // Use all available URLs provided to sniff the cluster.
 var urls []string
 urlsMap := make(map[string]bool)

 // Add all URLs provided on startup
 for _, url := range c.urls {
  urlsMap[url] = true
  urls = append(urls, url)
 }
 c.mu.RUnlock()

 // Add all URLs found by sniffing
 c.connsMu.RLock()
 for _, conn := range c.conns {
  if !conn.IsDead() {
   url := conn.URL()
   if _, found := urlsMap[url]; !found {
    urls = append(urls, url)
   }
  }
 }
 c.connsMu.RUnlock()

 if len(urls) == 0 {
  return errors.Wrap(ErrNoClient, "no URLs found")
 }

 // Start sniffing on all found URLs
 ch := make(chan []*conn, len(urls))

 ctx, cancel := context.WithTimeout(parentCtx, timeout)
 defer cancel()

 for _, url := range urls {
        // sniffNode 方法使用http，请求了对应的url，将结果封装后返回
  go func(url string) { ch <- c.sniffNode(ctx, url) }(url)
 }

 // Wait for the results to come back, or the process times out.
 for {
  select {
  case conns := <-ch:
   if len(conns) > 0 {
    c.updateConns(conns)
    return nil
   }
  case <-ctx.Done():
   if err := ctx.Err(); err != nil {
    switch {
    case IsContextErr(err):
     return err
    }
    return errors.Wrapf(ErrNoClient, "sniff timeout: %v", err)
   }
   // We get here if no cluster responds in time
   return errors.Wrap(ErrNoClient, "sniff timeout")
  }
 }
}
```
