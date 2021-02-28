---
layout: post
title:  "实现自定义 gRPC Resolver 遇到的问题"
tags: gRPC golang
---

在正式开始之前，先做一下铺垫，我们知道 `go` 里面是允许任何**可比较**的类型作为 `map` 的 key，包括布尔，数值，字符串，指针，`channel`，`interface` 以及只包含这些类型的 `struct` 和 `array` 类型，而不能被比较的类型，包括`slice`，`map`，`function`就不可以用来作为 `map` 的 key。需要注意的是这里说的**可比较**，就是使用操作符 `==`。假如一个包含指针成员的 `struct` 作为 `map` 的 key，那么只要指针本身的地址发生了变化，即使指针所指向的内容没有发生变化，那么作为 `map` 的 key 时，就是两个不相同的 `key`。比如，
```golang
package main

import (
    "fmt"
    "strconv"
)

type Bar struct {
    Num int
}

func (b *Bar) String() string {
    return strconv.Itoa(b.Num)
}

type Foo struct {
    Num int
    B   *Bar
}

func main() {
    m := map[Foo]int{}

    f0 := Foo{Num: 1, B: &Bar{Num: 2}}
    m[f0] = 0

    f1 := Foo{Num: 1, B: &Bar{Num: 2}}
    m[f1] = 1

    fmt.Println(m)
}
// output:
// map[{1 2}:0 {1 2}:1]
```

正式开始，我们知道 `gRPC` 支持自定义 `Resolver` 来实现自定义的服务发现机制，自定义 `Balancer` 来实现自定义的负载均衡策略。

假如我们要实现一个基于权重的负载均衡策略，通常这么做：
* 先实现一个 `Resolver` 获取目标服务的实例信息，服务实例信息中包含了目标节点的地址和权重信息。
gRPC 用来表示 `Resolver` 解析的服务实例信息的定义如下：
```golang
type Address struct {
    // Addr is the server address on which a connection will be established.
    Addr string

    // Attributes contains arbitrary data about this address intended for
    // consumption by the load balancing policy.
    Attributes *attributes.Attributes

    // 省略了其他不重要的字段
}
```
其中，`Attributes` 字段可以用来保存负载均衡策略所使用的信息，比如权重信息。

* 再实现一个 `Balancer` 来根据 `Resolver` 提供的实例权重来做负载均衡。
`gRPC` 提供了一个 `baseBalancer` 封装了一些通用的更新连接状态的逻辑，同时抽象了一个 `Picker` 的接口，可以简单理解为它是用来定义如何从 `Resolver` 提供的服务实例中选择一个可用实例。所以，通常如果没有特殊需求的话，我们只需要实现一个 `Picker` 就可以，将实现的 `Picker` 和 `baseBalancer` 组合就可以生成一个新的 `Balancer`。而这次遇到的问题就是出在这个 `baseBalancer` 和 `Address` 定义上。

在 `gRPC v1.35.0` 之前的版本，`baseBalancer` 对连接的管理是这样实现的：
```golang
func (b *baseBalancer) UpdateClientConnState(s balancer.ClientConnState) error {
	// 省略部分代码...

	// addrsSet is the set converted from addrs, it's used for quick lookup of an address.
	addrsSet := make(map[resolver.Address]struct{})
	for _, a := range s.ResolverState.Addresses {
		addrsSet[a] = struct{}{}
		if _, ok := b.subConns[a]; !ok {
			// a is a new address (not existing in b.subConns).
			sc, err := b.cc.NewSubConn([]resolver.Address{a}, balancer.NewSubConnOptions{HealthCheckEnabled: b.config.HealthCheck})
			if err != nil {
				logger.Warningf("base.baseBalancer: failed to create new SubConn: %v", err)
				continue
			}
			b.subConns[a] = sc
			b.scStates[sc] = connectivity.Idle
			sc.Connect()
		}
	}
	for a, sc := range b.subConns {
		// a was removed by resolver.
		if _, ok := addrsSet[a]; !ok {
			b.cc.RemoveSubConn(sc)
			delete(b.subConns, a)
			// Keep the state of this sc in b.scStates until sc's state becomes Shutdown.
			// The entry will be deleted in UpdateSubConnState.
		}
	}

	// 省略一些不重要的代码...
	return nil
}
```
每当 `Resolver` 解析的地址列表有更新时就会回调 `balancer.UpdateClientConnState` 方法，通知 `Balancer` 对底层的连接进行更新，主要就是先对新出现的地址建立连接，然后再关闭已经不存在的地址对应的连接。我们看到，`baseBalancer` 使用 `subConns` 保存地址与连接对应关系，它的定义是 `map[resolver.Address]balancer.SubConn`，使用了 `resolver.Address` 作为 `map` 的 `key`。问题来了，因为我们使用了 `resolver.Address` 中的 `Attributes` 来保存权重值，而 `Attributes` 是一个指针类型，如果 `Resolver` 在每次解析地址发生变更，即使地址和权重都未发生变化时，都创建一个新的 `resolver.Address` 的话，这样就会导致 `baseBalancer` 认为这是一个新的地址，从而会造成 `baseBalancer` 对相同的地址再重新建立连接，然后关掉该地址对应的上一条连接，当 `gRPC client` 要访问的目标服务有大量节点时，这就会导致每次 `Resolver` 解析地址有变更时（哪怕可能只有一个节点新增或者减少），都会导致大量的连接断开和重连。

对于这个问题，有两个解决办法。
1. 自己实现一个 `Balancer`，把 `resolver.Address.Addr` 作为 `Map` 的 `Key`，权重不参与比较，可以单独保存；
2. 在实现 `Resolver` 时，缓存一份上一次解析的 `Address` 列表结果，当有新的地址解析结果时，先和上一次解析结果的 `Address` 列表进行对比，只有当 `Addr` 或者权重有变更时，才创建一个新的 `resolver.Address`，否则还是使用一次创建的 `Address` 对象，避免 `Attributes` 地址发生变化。

考虑到方法2实现上相对简单，所以我们采用了方法2。

但是当我们把 `gRPC` 升级到 `v1.35.0` 及以上的版本时，发现服务实例的权重信息丢失了。翻看 Release Notes 找到这个 [PR #4024](https://github.com/grpc/grpc-go/pull/4024)，这个是为了修复上面说的问题，在使用 `resolver.Address` 作为 `baseBalancer.subConns map` 的 `key` 之前，先去掉了 `Attributes` 字段（设置为 `nil`）。 但是，我们实现的 `Picker` 是通过 `baseBalancer.subConns` 的 `key` 来获取实现权重，然后实现按权重选择实例的。但是目前 `Attributes` 被去掉了，导致权重丢失。只能回到方案1的解决办法上了。

希望我把问题讲清楚了，如果你也遇到了，希望能有帮助。
