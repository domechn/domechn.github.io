---
title: 记一次 WebSocket 连接泄露排查
subtitle: websocket-leak
date: 2019-12-29 16:34:34
tags:
- issues
- websocket
---

之前重构了某个 `WebSocket` 服务，在运行了一段时间之后有同事反应该服务卡顿，时常获取不到数据。

于是我尝试开始复现该情况。

打开浏览器跳转到相应的页面后，发现数据立刻就加载了出来，但是多次刷新页面之后，就开始出现无法加载数据的情况。打开控制台，发现报了如下的错误。

``` bash
WebSocket connection to 'ws://xxxxx/' failed: Error in connection establishment: net::ERR_CONNECTION_FAILED
```

一般来说出现该问题可能是因为网络问题或者连接超时引起的，在排除了自身网络问题的前提下，我调大了 WebSocet 的连接超时时间。但还是出现了上述的情况。

接着我猜想是不是我们的长连接网关出了问题。于是我查看了网关的日志，发现网关压根就没接收到出现 `ERR_CONNECTION_FAILED` 错误的连接。

于是我确定是之前重构的项目问题。

我开始查看 `Pod` 的日志，发现在连接建立成功并断开的时候没有将日志打印出来。

> 该项目用了 `echo` 框架，并使用了 `LogMiddleware` ，它会在每条请求结束后将请求信息打印出来。

那应该就是在客户端断开 `WebSocket` 连接的时候，服务端并没有断开，导致了 `WebSocket` 连接泄露。

进入到 `Pod` 中查看连接数

``` bash
netstat -na|grep ESTABLISHED|wc -l
> 19
```

成功建立连接之后

``` bash
netstat -na|grep ESTABLISHED|wc -l
> 23
```

客户端退出之后

``` bash
netstat -na|grep ESTABLISHED|wc -l
> 21
```

显然是连接泄露了。浏览器会对同一域名的连接数做限制，所以有可能是因为连接泄露导致连接数过大，使之后的连接无法建立才出现的 `ERR_CONNECTION_FAILED` 的情况。

于是开始定位代码中的问题，以下是大概的代码逻辑。

这是一个和 `K8S` 管理平台相关的项目。服务会监听 `Pod` 的状态变化，并通过 `WebSocket` 将变化结果通知给前端。

``` go
func handler(c echo.Context) error {
    // ...
    // 使用 request.Context() 作为全局 context
 ctx := c.Request().Context()

 websocket.Handler(func(ws *websocket.Conn) {
  defer ws.Close()
  if err := watchPodStatus(ctx, param.Namespace, func(e watch.Event) (bool, error) {
    // 将变化发给前端
    websocket.Message.Send(ws, ...)
        }); err != nil {
   websocket.Message.Send(ws, err.Error())
  }
 }).ServeHTTP(c.Response(), c.Request())

 return nil
}
// WatchPodStatus
// name: deployment name
func watchPodStatus(ctx context.Context, namespace string, conditionFunc func(e watch.Event) (bool, error)) error {
    // ...
    // 这步会列出变化的 pods
    // 通过 conditionFunc 方法处理事件
    // 注：该方法是阻塞的，除非 ctx 被 cancel 或者 conditionFunc 返回 true 否则不会结束
 _, err = watchtools.UntilWithSync(ctx, listAndWatchFunction, &corev1.Pod{}, preconditionFunc, conditionFunc)
 return err
}
```

这是我使用了 `echo.Request().Context()` 作为 `watchtools.UntilWithSync` 的 context，原本设想的是在请求结束后 context 就会 `cancel` ，从而终止 `watch` 事件，并退出整个请求。

结果 `request.Context()` ，并没有在前端调用 `socket.close()` 的时候退出。导致 `watch` 也没有退出，所以 `defer ws.Close()` 也就无法触发，导致服务端无法结束该次请求。

在知道是该原因后问题也好解决了。以 `request.Context()` 作为父 `context` 创建一个新的 `context` 。使用一个 `goroutine` 监听 `WebSocketConnection` 的状态。（通过不停的 `conn.Receive()` ），因为该场景没有客户端向服务端发送数据的场景，所以该调用会被一直阻塞，直到客户端调用 `socket.close()` 之后， `conn.Receive()` 会接收到一个错误信息，这时再 `cancel()` 掉上述的 `context` 退出 `watch` 即可，以下是修改后的代码逻辑。

``` go
func handler(c echo.Context) error {
    // ...
    // 使用 request.Context() 作为全局 context
    ctx := c.Request().Context()
    cancelctx, cancel := context.WithCancel(ctx)

    websocket.Handler(func(ws *websocket.Conn) {
        go func(){
            for {
                // receive 是阻塞方法，如果没有收到消息就会一直等待
                // 收到错误说明连接断开了或者出现异常了，这是取消掉 ctx，watch 行为也会推出
                if err := websocket.Message.Receive(ws, &res); err != nil{
                    cancel()
                }
            }
        }()
    defer ws.Close()
    if err := watchPodStatus(cancelctx, param.Namespace, func(e watch.Event) (bool, error){
                // 将变化发给前端
                websocket.Message.Send(ws, ...)
            }); err != nil {
    websocket.Message.Send(ws, err.Error())
    }
    }).ServeHTTP(c.Response(), c.Request())

    return nil
}
```

完！
