---
slug: "push-server-summarize"
title: "推送服务开发过程的问题总结"
description: "在开发开发 push service 的过程中,有一大部分时间花在 websocket 的使用和连接管理上,期间遇到了一些问题和挑战,这里做一个总结"
lead: "在开发开发 push service 的过程中,有一大部分时间花在 websocket 的使用和连接管理上,期间遇到了一些问题和挑战,这里做一个总结"
date: 2022-06-10T09:19:42+08:00
lastmod: 2022-06-10T09:19:42+08:00
draft: false
weight: 98
contributors: ["Yusank"]
---

- [前言](#前言)
- [事件管理](#事件管理)
- [连接管理](#连接管理)
- [消息推送](#消息推送)
- [总结](#总结)

## 前言

push server 的开发是在做 goim 的前期的重头戏,其核心就是长链接(本次用 websocket)的管理和与业务相互结合.

websocket 算是一个比较久远成熟的长链接方案况且有成熟的框架(本项目使用的 [gorilla/websocket](https://github.com/gorilla/websocket))支持,按理来说,它的使用应该是比较简单的,但是它的使用过程中,有一些问题和挑战,这里做一个总结.

而遇到的问题大部分是在使用和与业务的整合上,大致分为一下几点:

- 事件管理(如 ping pong,心跳,消息等)
- 连接管理
- 消息推送

## 事件管理

在建立起连接之后,需要对一些 ws 消息类型做不同的处理,如 ping/pong 时,连接管理层面刷新一些连接最后活跃时间已确保连接可用,而业务层面需要对该连接对应的用户
的 last_online_time 信息进行刷新动作,又比如 close 事件也是连接管理层面和业务层面各自需要做处理.

虽说框架提供了注册事件处理器的机制,但是不足以满足需求.连接层是纯粹管理长链接不依赖业务逻辑（且连接层与业务是分开的项目代码），因此没办法在连接层直接引入逻辑层的逻辑去注册相关处理函数。而业务层虽说可以引入连接层的逻辑，但是不同业务处理的方式不一样很难统一管理。

以下为原始注册方法，只能注册单个方法，如果需要有多个处理函数则需要手动嵌套：

```go
// 连接层注册连接层的事件处理器
conn.SetCloseHandler(func(code int, text string) error {
    wc.cancelWithError(nil)
    message := websocket.FormatCloseMessage(code, "")
    _ = wc.conn.WriteControl(websocket.CloseMessage, message, time.Now().Add(time.Second))
    return nil
})

conn.SetPingHandler(func(message string) error {
    err := c.WriteControl(websocket.PongMessage, []byte(message), time.Now().Add(time.Second))
    if err == nil || err == websocket.ErrCloseSent {
        return nil
    }

    return err
})
```

{{< alert icon="👉" context="info" text="同样的事件需要在不同的逻辑层做不同的处理且不能侵入到连接层。" />}}

为了解决这个问题，很明显框架提供的原始的结构体的能力是不够，需要进行封装扩展。

```go
type WebsocketConn struct {
    conn *websocket.Conn // 原始连接

    // ctx control
    ctx    context.Context
    cancel context.CancelFunc

    uid          string
}


func WrapWs(ctx context.Context, c *websocket.Conn, uid string) *WebsocketConn {
    if ctx == nil {
        ctx = context.Background()
    }
    ctx2, cancel := context.WithCancel(ctx)
    wc := &WebsocketConn{
        conn:      c,
        ctx:       ctx2,
        cancel:    cancel,
        uid:       uid,
    }

    wc.conn.SetCloseHandler(func(code int, text string) error {
        wc.cancelWithError(nil)
        message := websocket.FormatCloseMessage(code, "")
        _ = wc.conn.WriteControl(websocket.CloseMessage, message, time.Now().Add(time.Second))
        return nil
    })

    wc.conn.SetPingHandler(func(message string) error {
        err := c.WriteControl(websocket.PongMessage, []byte(message), time.Now().Add(time.Second))
        if err == nil || err == websocket.ErrCloseSent {
            return nil
        }

        return err
    })

    return wc
}

// 通过该方法注册业务的处理方法
func (w *WebsocketConn) AddCloseAction(f func() error) {
    cf := w.conn.CloseHandler()
    w.conn.SetCloseHandler(func(code int, text string) error {
        err := cf(code, text)
        if err == nil {
            return f()
        }

        return err
    })
}

// 通过该方法注册业务的处理方法
func (w *WebsocketConn) AddPingAction(f func() error) {
    pf := w.conn.PingHandler()
    w.conn.SetPingHandler(func(appData string) error {
        err := pf(appData)
        if err == nil {
            return f()
        }

        return err
    })
}
```

在连接层将原始 conn 进行一层封装，逻辑层能取到封装后的连接，可根据业务需求注册任意数量的事件处理器。而连接层在封装连接时就将连接层需要的处理方法注册进去了。

现在看一下业务层的使用方式：

```go
// 建立连接并封装后注册业务处理函数
wc.AddPingAction(func() error {
    return app.GetApplication().Redis.SetEX(context.Background(),
        consts.GetUserOnlineAgentKey(uid), app.GetAgentIP(), consts.UserOnlineAgentKeyExpire).Err()
})
// 建立连接并封装后注册业务处理函数
wc.AddCloseAction(func() error {
    ctx2, cancel := context.WithTimeout(ctx, time.Second)
    defer cancel()
    return app.GetApplication().Redis.Del(ctx2, consts.GetUserOnlineAgentKey(uid)).Err()

})
```

这样就把业务和连接管理拆分开互不影响，连接层不关心上层业务怎么处理，只关心连接的正常可用即可。

## 连接管理

连接管理也是花心思比较多的模块，由于 websocket 的连接不同于普通的连接，因此没办法用普通的连接池来管理（其实我已经有一套可用的连接池的方案了[yusank/conn-pool](https://github.com/yusank/conn-pool)，可惜了）。

websocket 的每个连接都是唯一的且需要根据一些唯一的值（UID，sessionID 等）来拿到对应的连接，这些连接也不是连接池初始化出来的，是外部传进去让连接池去维护这些连接。

同时 websocket 连接随时可能断开，这种断开可能主动的也可能是连续几个心跳周期没有收到客户端的心跳包从而服务端主动断开的，不管是那种情况，在连接层需要有个 goroutine 去维护这个连接的可用性。在连接需要断开时，停止维护并从连接池删除该连接。

为了确保 `WebsocketConn` 的纯洁性，在维护连接模块里定义一个简单的`连接`的概念,把 `WebsocketConn` 封装起来。

```go
type idleConn struct {
    *WebsocketConn
    p        *namedPool // 连接池
    stopChan chan struct{}
}

// close is different form stop
// close is closes the connection and delete it from pool
// stop is a trigger to stop the daemon then call the close
func (i *idleConn) close() {
    _ = i.Close() // i.WebsocketConn.Close()
    i.p.delete(i.Key()) // i.WebsocketConn.Key() => uid
}

func (i *idleConn) stop() {
    i.stopChan <- struct{}{}
}

func (i *idleConn) daemon() {
    ticker := time.NewTicker(config.Interval * time.Second)
loop:
    for {
        select {
        case <- ticker.C:
             // check last ping time and decide whether stop daemon.
        case <-i.ctx.Done():
            // WebsocketConn ctx cancelled
            log.Error("conn done", "key", i.Key())
            break loop
            //  stop() called
        case <-i.stopChan:
            log.Info("conn stop", "key", i.Key())
            break loop
        case data := <-i.writeChan:
            i.writeToClient(data) // write msg to client
        }
    }

    log.Info("conn daemon exit", "key", i.Key())
    i.close()
}
```

`idleConn` 能力相对纯粹，维护连接可用性和特殊情况下从连接池剔除当前连接。

下面看一下连接池的定义和实现：

```go
type namedPool struct {
    *sync.RWMutex
    m map[string]*idleConn
}

func newNamedPool() *namedPool {
    p := &namedPool{
        RWMutex: new(sync.RWMutex),
        m:       make(map[string]*idleConn),
    }

    return p
}

// 添加连接
func (p *namedPool) add(c *WebsocketConn) {
    select {
    case <-c.ctx.Done():
        return
    default:
        if c.Err() != nil {
            return
        }
    }

    p.Lock()
    defer p.Unlock()
    // 若已存在 则先停止上一个 daemon
    i, loaded := p.m[c.Key()]
    if loaded {
        i.stop()
    }

    i = &idleConn{
        WebsocketConn: c,
        stopChan:      make(chan struct{}),
        p:             p,
    }

    // TODO: 使用可控大小的 goroutine 池，防止goroutine 泄露
    go i.daemon()
    p.m[c.Key()] = i
}

func (p *namedPool) get(key string) *WebsocketConn {
    p.RLock()
    i, ok := p.m[key]
    p.RUnlock()

    if ok {
        // 拿连接时 先判断是否已失效
        select {
        case <-i.ctx.Done():
            i.stop()
        default:
            if i.Err() != nil {
                i.stop()
                return nil
            }
            return i.WebsocketConn
        }
    }

    return nil
}

// 需要读取所有连接做全量推送的需求
func (p *namedPool) loadAllConns() chan *WebsocketConn {
    p.Lock()
    defer p.Unlock()

    ch := make(chan *WebsocketConn, len(p.m))
    for _, i := range p.m {
        ch <- i.WebsocketConn
    }

    close(ch)
    return ch
}

func (p *namedPool) delete(key string) {
    p.Lock()
    defer p.Unlock()
    delete(p.m, key)
}

```

经过 `idleConn` 和 `namedPool` 配合，连接管理算是达到预期的要求了且通过了测试，后期可以做一些优化项。

## 消息推送

在解决了非业务层的各种问题后，现在只剩下消息推送的问题了。消息推送最开始其实直接调用的框架提供的原生方法 `WriteMessage`,即需要推送时，从连接池拿到对应的连接然后直接调用写消息的方法，但是有大量消息需要写入时可能会出现消息顺序异常的问题，因此在 `WebsocketConn` 层屏蔽了原生的方法，并添加一个消息 `channel` 来控制并发带来的消息顺序异常问题。

新增消息 `channel` :

```gotype WebsocketConn struct {
    conn *websocket.Conn

    ctx    context.Context
    cancel context.CancelFunc

    uid          string
    // 消息 channel
    writeChan    chan []byte
    // 无法写到客户端时回调该方法 上层一般会进行重试或写到 mq 待会儿进行处理
    onWriteError func()
    err          error
}
```

新的写消息方法：

```go
func (w *WebsocketConn) Write(data []byte) error {
    select {
    case w.writeChan <- data:
        return nil
    default:
    }

    // 尝试等待一小段时间，如果这段时间内写入失败则直接报错，这种情况下大概率是服务端与客户端的连接异常，消息阻塞了
    timer := time.NewTimer(time.Millisecond * 10)
    select {
    case w.writeChan <- data:
        return nil
    case <-timer.C:
        return ErrWriteChanFull
    }
}
```

写入到 `writeChan` 后，由上述 `idleConn.daemon` 方法读取 `channel` 内消息后，调用以下方法来正式写入的ws 上：

```go
func (w *WebsocketConn) writeToClient(data []byte) {
    _ = w.conn.SetWriteDeadline(time.Now().Add(time.Second))
    err := w.conn.WriteMessage(websocket.TextMessage, data)
    if err != nil {
        w.onWriteError()
        return
    }
}
```

至此，消息的推送也解决了已遇到的问题。

## 总结

本文记录了在开发 goim 的推送服务(长连接服务)时遇到的一些问题和解决方案，方便理解在看相关代码时其背后的思考。

内容分为三个部分：

- websocket 连接中的一些事件处理和如何与业务关联起来
- websocket 连接池的维护
- 在聊天系统中确保消息推送的顺序
