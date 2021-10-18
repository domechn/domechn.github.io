---
title: 如何用Redis实现一个指定时间的限速器
subtitle: redis_limit
date: 2019-02-28 14:53:58
tags:
- redis
---

使用Redis的`Incr`可以很容易的实现一个限速器

在redis的官方文档中也有详细的示例

```c
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
current = GET(keyname)

IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
END

IF current == NULL THEN
    MULTI
        INCR(keyname, 1)
        EXPIRE(keyname, 1)
    EXEC
ELSE
    INCR(keyname, 1)
END

PERFORM_API_CALL()
```

### 举个例子

现有一个服务，1分钟内只能接受一个用户不超过10次的请求，这时我们可以将用户的ip地址设置为key，用户每次时程序去redis中获取该key的值，如果大于等于10则返回错误，否则给key对应的value+1即可，如果value为0，那么再将该key设置1分钟的过期时间。

---

**但是现在有一个需求，我们可以在一个指定的时间内给用户推送一条消息，但是要求用户每分钟内只能接受1条消息，每小时内接受的消息不超过5条，一天内接受的消息不超过10条。**

比如，我现在向这个接口提交了一条数据，要求在2019-11-11 11:11:11时向一个用户发送一条数据，那么当我再提交一条数据，要求在2019-11-11 11:11:12是向同样的用户发送一条数据，那么接口就会返回错误。

此时，仅使用`Incr`是无法满足该需求的。

### 解决方法

使用`set`或者`zset`将用户的`ip+发送日期`作为key,发送时间转换为当天的秒数作为value，插入到`set`或者`zset`中，每次向用户提交信息时，可以获取到`set`中的所有发送时间，然后再一一比对，如果不满足条件就返回错误。

以下是伪代码实现

```
function RateLimit(ip,sendtime){
  // 根据发送时间得到发送的日期
  sendDate = getSendDate(sendtime)
  // 获取发送时间和当天0点之间的秒数差值
  second = getSendSecond(sendtime)
  // 列出该发送日期中的所有发送时间
  secondList = listSeconds(ip + sendDate)
  // 检验成功
  if check(second,secondList) {
    // 将发送时间添加到缓存
    return addToList(ip + sendDate,second)
  }
  return false
}
```

以下是golang的实现

```go
func RateLimit(ctx context.Context, ip string, sendTime time.Time) error {
    sendTime = sendTime.UTC()
 if sendTime.Before(time.Now()) {
  return nil
 }
 zeroStr := sendTime.Format("2006-01-02")
 key := ip + zeroStr
 zero, _ := time.Parse("2006-01-02", zeroStr)
    // 获取发送时间距当天时间的秒数
 second := int(sendTime.Sub(zero).Seconds())
 ress, err := cache.SMembers(ctx, key)
 if err != nil {
  return err
 }
    // 处理返回参数，将[]string转换为[]int
    var sends []int
 for _, v := range ress {
  if vv, err := strconv.Atoi(v); err == nil {
   sends = append(sends, vv)
  }
 }
 if len(sends) >= 10 {
  return errors.ErrMsg1DayLimit
 }

 var expire bool

 if len(sends) == 0 {
  expire = true
 }

 var hourCount int

 for _, s := range sends {
  if math.Abs(float64(second-s)) < 60 {
   return errors.ErrMsg1MinuteLimit
  }
  if math.Abs(float64(second-s)) < 3600 {
   hourCount++
  }
 }
 if hourCount > 5 {
  return errors.ErrMsg1HourLimit
 }
    // 异步更新
    go func() {
        // 开启事务
  t := cache.Pipeline()
  defer t.Close()
  t.SAdd(context.Background(), key, []byte(strconv.Itoa(second)))
  if expire {
   ttl := zero.Add(time.Hour*24).Sub(time.Now()).Seconds()
   if ttl <= 0 {
    t.Rollback()
    return
   }
   t.Expire( key, int64(ttl))
  }
  t.Commit()
 }()
    return nil
}
```

以上就可以实现一个指定时间的限速器。
