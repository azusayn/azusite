---
title: 设计模式
date: 2026/9/8 19:44:00
categories:
- Golang
tags:
- Golang
---

- Singleton Pattern

```go
type Runtime struct {}

var (
    runtime *Runtime
    runtimeOnce sync.once
)

func NewRuntime() *Runtime {
    runtimeOnce.Do(func() {
        runtime = &Runtime{}
    })
    return runtime
}
```

- Option Pattern

```go
type Runtime struct{}

type Option func(*Runtime)

func NewRuntime(opts ...Option) *Runtime {
    r := &Runtime{}
    for _, opt := range opts {
        opt(r)
    }
    return r
}
```

- Decorator

```go

type Item interface {
    Cost()
}

type Coffee struct {}

func (c *Coffee) Cost() int {
    return 10
}

type Sugar struct {
    item Item
}
 
func (s *Sugar) Cost() int {
    return s.item.Cost() + 5
}

func main() {
    var i Item = Coffee{}
    i = Sugar{item: i}

    fmt.Printf("cost: %d\n", i.Cost())
}
```
