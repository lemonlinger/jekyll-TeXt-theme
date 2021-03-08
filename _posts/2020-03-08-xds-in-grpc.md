---
layout: post
title:  "浅淡 xDS 协议在 gRPC 中的应用"
tags: gRPC xDS Envoy
---

## `Service Config`
当我们实现了一个微服务，为了让客户端更『智能』的和服务端进行交互，有些服务本身的配置是需要让客户端知道的，比如：

* 期望客户端对服务端的实例所采用的负载均衡策略；
* 当调用失败时，期望客户端采用什么样的重试策略，返回什么错误码需要重试，重试的次数，避让的策略等等；
* 期望客户端调用某些接口时需要遵守的超时时间。

以上提到的这些配置，在 `gRPC` 中有一个对应的概念叫 `ServiceConfig`，具体包含哪些内容，可以参考 [`Service Config in gRPC`](https://github.com/grpc/grpc/blob/master/doc/service_config.md) 

`Service Config` 主要在两个方面影响到 `gRPC Client` 的行为：

* 在创建 Client Stream 时，应用 `Service Config` 中的 `MethodConfig`，比如超时，重试策略等；
* 在 `gRPC Client` 选择负载均衡策略时，创建并应用 `ServiceConfig` 中指定的负载均衡器及配置。

那么 `gRPC` 是怎么让客户端在调用服务端接口之前获得这些 `ServiceConfig` 呢？有两种方式：

1. 静态方式，即使用 `gRPC` 提供的 `DialOption` `WithDefaultServiceConfig`，在创建 `Client` 时指定，一般是作为当 `name resolver` 没有返回 `ServiceConfig` 时的默认配置；
2. 动态方式，即通过 `name resolver` 来动态获取 `ServiceConfig`。`name resolver` 是可以在返回目标地址列表的同时返回对应目标服务的 `ServiceConfig`.

在 `gRPC` 引入 `xDS` 协议之前，除非自定义 `name resolver`，否则，只能通过内置的 `DNS name resolver` 来动态获取 `ServiceConfig`，原理上是通过 [`DNS TXT`](https://github.com/grpc/proposal/blob/master/A2-service-configs-in-dns.md) 记录来保存 `ServiceConfig` 信息。需要注意的是 [`grpclb`](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md) 并不是 `name resolver`，而是一个 `balancer`，虽然他同时也干了 `resolver` 干的事，但是 `ServiceConfig` 还是通过 `DNS` 来获取的。

`grpclb` 可以说是 `gRPC` 对自定义负载均衡（或者说服务发现）服务的一次探索，使用了自定义的接口协议，虽然功能比较粗糙，但是也算一次尝试。由于 `xDS` 逐渐成为了负载均衡领域服务发现的事实标准，所以，`gRPC` 也开始转向支持 `xDS` 协议，`grpclb` 协议就处于一个废弃的状态了。

## `xDS`
`xDS` 起源于 `Envoy`，是 `Envoy` 通过查询一个或多个管理服务器（控制面）来发现其各种动态资源的协议，这些发现服务及其相应的 `API` 统称为 `xDS`，更多的细节可以参考 [`Envoy` 的官方文档](https://www.envoyproxy.io/docs/envoy/latest/api/api)。

对于 `Envoy` 来说，`xDS` 的配置一部分是作用于 `Envoy` 与 `downstream` 的交互上，一部分是作用于与 `upstream` 的交互上；而 `gRPC` 作为一个 `RPC` 的库，那就相当于分别对应于 `Client Side` 与 `Server Side` 的实现。

## `Client Side` 

当前 `xDS` 在 `gRPC Client` 上的实现架构如下：
![grpc-client-arch.png](https://github.com/grpc/proposal/raw/master/A27_graphics/grpc_client_architecture.png)

`xDS` 本身就包含了服务发现和负载均衡的概念，因此，在 `gRPC Client` 上同时对应了 `resolver` 和 `balancer`，`xDS Resolver` 与 `xDS Balancer` 共享同一个 `xDS Client`，`xDS Resolver` 通过 `xDS` 中的 `LDS` 和 `RDS` 来实现服务发现的功能，通过 `CDS` 和 `EDS` 来实现负载均衡的逻辑。不过，要注意的是 `RDS` 并不会直接返回目标服务的地址列表，而是要等 `EDS` 完成之后才会得到地址列表。`EDS` 支持按优先级和权重来分配流量，不过，目前权重只支持 `Locality` 级别的，对于同一个 `Locality` 的 `Endpoint` 还是默认使用 `round-robin`。

在 `gRPC` 的底层实现中，以上提到 `xDS` 资源都会先转化为 `ServiceConfig` 然后还是通过原来的更新机制作用于 `Client Stream`  和负载均衡策略上，更多详细的详细，请参考 [A27-xds-global-load-balancing](https://github.com/grpc/proposal/blob/master/A27-xds-global-load-balancing.md)。

2021 年开始，`gRPC` 继续在 `xDS` 协议上进行推进，目前已经在进程中的有[熔断](https://github.com/grpc/proposal/blob/master/A32-xds-circuit-breaking.md)，[故障注入](https://github.com/grpc/proposal/blob/master/A33-Fault-Injection.md)，[支持 `HTTP Filter`](https://github.com/grpc/proposal/blob/master/A39-xds-http-filters.md) 等。如果读者了解 `Envoy` 的话，应该知道 `Envoy` 丰富的 [`HTTP Filter`](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/http_filters) 功能，而且支持用户自定义。虽然，目前 `gRPC` 对 `HTTP Filter` 功能的支持只是刚起步，但是值得期待。

## `Server Side`
`xDS` 在 `gRPC Client` 方向上的进展已经持续一年多了，而在 `gRPC Server` 方向才刚刚开始。

如前所述，`xDS` 的一部分配置对应的是 `gRPC Client`，还有一部分对应的是 `gRPC Server` 部分的功能需求，比如 `Listen` 的端口，`TLS` 的配置，鉴权，限流等配置，在 `Envoy` 中这部分配置也是可以通过 `xDS` 协议进行动态发现的。对于 `gRPC Client`，由于已经有了 `Resolver` 和 `Balancer` 的扩展机制，通过实现对应 `xDS Resolver` 和 `Balancer` 就可以轻易完成 `xDS` 协议在 `gRPC Client` 上的应用，而 `gRPC Server` 的扩展机制就只有 `interceptor`，但是实现一个类似的 `xDS interceptor` 又不能满足 `gRPC Server` 动态获取 `Server` 端配置，并在配置更新时得到通知的功能需求。`gRPC` 给出的方案是在原有的 `gRPC Server` 基础上实现一个 `XdsServer`，实现 `xDS` 的逻辑，并且提供原 `Server` 相同的能力，更多的细节请参考 [A36-xds-for-servers](https://github.com/grpc/proposal/blob/master/A36-xds-for-servers.md

P.S. 如果想了解 `gRPC` 最近的动向，可以关注 `gRPC Proposal` 的 [`repo`](https://github.com/grpc/proposal)，`gRPC` 的功能变更都会先通过提交 `gRFC` 文档，评审通过后才会进入实现阶段。
