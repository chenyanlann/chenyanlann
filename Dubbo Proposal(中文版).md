### Apache Dubbo GSoC2022 Proxyless Mesh support

#### 1. 方案设计

#### Objective

通过不同语言版本的 Dubbo3 SDK 直接实现 xDS 协议解析，实现 Dubbo 与 Control Plane 的直接通信，进而实现控制面对流量管控、服务治理、可观测性、安全等的统一管控，规避 Sidecar 模式带来的性能损耗与部署架构复杂性。

#### Proposal

[xds-2](https://github.com/apache/dubbo-awesome/blob/master/images/mesh-xds-2.png)

在 Dubbo3 Proxyless 架构模式下，Dubbo 进程将直接与控制面通信，Dubbo 进程之间也继续保持直连通信模式，我们可以看出 Proxyless 架构的优势：

- 没有额外的 Proxy 中转损耗，因此更适用于性能敏感应用
- 更有利于遗留系统的平滑迁移
- 架构简单，容易运维部署
- 适用于几乎所有的部署环境

#### Architecture

[xds-1](../images/mesh-xds-1.png)

#### Details

1. xDS 官方说明

https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol

● Listener Discovery Service (LDS): Returns Listener resources. Used basically as a convenient root for the gRPC client's configuration. Points to the RouteConfiguration.
● Route Discovery Service (RDS): Returns RouteConfiguration resources. Provides data used to populate the gRPC service config. Points to the Cluster.
● Cluster Discovery Service (CDS): Returns Cluster resources. Configures things like load balancing policy and load reporting. Points to the ClusterLoadAssignment.
● Endpoint Discovery Service (EDS): Returns ClusterLoadAssignment resources. Configures the set of endpoints (backend servers) to load balance across and may tell the client to drop requests.

2. 协议基础：protobuf + gRPC

实现方案 1：使用 io.grpc 的依赖手动创建原生 gRPC 服务
实现方案 2：使用 Dubbo 3.0 的新协议

3.  测试环境控制平面 mock

Envoy 提供的 Java 实例控制平面 https://github.com/envoyproxy/java-control-plane

4.  Java 版协议支持

Envoy 提供的 Java 版控制平面 API https://search.maven.org/search?q=g:io.envoyproxy.controlplane

协议可利用内容

● LDS： envoy.api.v2.Listener
● RDS： envoy.api.v2.RouteConfiguration
● CDS： envoy.api.v2.Cluster
● EDS： envoy.api.v2.ClusterLoadAssignment

LDS  

Listener 发现协议，在 sidecar 部署模式下用户配置本地监听器，由于 Dubbo 去接入 xDS 本身就是一个 proxy-less 的模式，所以 LDS 很多的配置是不需要考虑的。
针对 LDS 协议，需要做的有获取适配的监听器用于获取路由配置。
对于 LDS 协议，有些控制平面并不能保证获取到的监听器是唯一的，所以需要兼容获取到多个监听器时的配置情况。

RDS  

Route 发现协议，通过路由配置可以对请求的地址进行匹配，经过一定的策略后获取对应的上游 Cluster 集群。
针对 RDS 协议，可以在基于 Listener 监听器的配置，获取到一组对应的路由协议。然后根据服务自省框架对应的服务名筛选出对应的 Cluster 集群。
对于 RDS 协议，有些控制平面并不能保证获取到的l路由组是唯一的，所以需要兼容获取到多个路由组时的配置情况。
在 RDS 的结果中包括了流控还有重试的配置，这些可以作为路由配置和负载均衡部分的默认配置来源。

CDS  

Cluster 发现协议，可以获取到对应集群内的配置（如超时时间、重试限制、元数据信息等），是基于 RDS 获取的。
CDS 的结果可以作为路由配置和负载均衡部分的配置来源。

EDS  

Endpoint 发现协议，可以获取到集群中对应节点信息，即是服务间通信的底层节点信息。
通过 EDS 的结果，可以获取到对应的节点地址信息、权重信息、健康信息，进而组成服务自省框架最基础的实例信息。

接入模型设计  

采用注册中心的模式对接，配置通过 ServiceInstance 运行时配置传出，通过 Invoker 传递到服务调用时。

服务发现逻辑  

xDS 接入以注册中心的模式对接，节点发现同其他注册中心的服务自省模型一样，对于 xDS 的负载均衡和路由配置通过 ServiceInstance 的动态运行时配置传出，在构建 Invoker 的时候将配置参数传入配置地址。


服务调用逻辑  

在 RPC 调用链路中，当获取负载均衡策略时根据 Invoker 的参数信息构建负载均衡信息（适配 EDS 的权重信息），对于集群的重试逻辑配置则工具 RDS 和 CDS 的配置进行修改。

#### 2. 个人技能描述

我是北京邮电大学的一名研究生，课余时间和假期有充裕的时间。我对学习和探索计算机的不同领域非常感兴趣，在过去的一段时间里，我很幸运地参加了开源社区，并成为了一些项目的贡献者。我希望更多地了解和了解开源社区，为Dubbo做出贡献。我有扎实的计算机和网络基础知识，熟悉java编程，了解Service Mesh和RPC。我喜欢学习知识，分析问题，希望与Dubbo社区一起成长，也愿意花足够的时间和精力。

https://github.com/chenyanlann

#### 编程语言

熟悉JAVA编程，包括Java基础，Java容器，Java并发，JVM，JNI等，阅读过《Java核心技术》，《Effective Java中文版》，《深入理解Java虚拟机》等书籍。Dubbo是用JAVA开发的分布式框架。

#### 相关计算机基础知识

了解网络协议，操作系统，数据结构等计算机基础知识。学校的主要课程包括程序设计实践，计算机网络，数据结构与算法等，RPC是一种通过网络从远程计算机程序上请求服务，基于底层网络传输协议，如TCP或UDP，为通信程序之间携带信息数据。在OSI网络通信模型中，RPC跨越了传输层和应用层。RPC使得开发包括网络分布式多程序在内的应用程序更加轻易。

#### 了解Service Mesh

我在公司使用过Mesh服务来完成服务之间的通信，包括出流量Mesh和入流量Mesh，通过Mesh实现了请求流量的分流和定向。服务网格是一种可配置的、低延迟的基础设施层，旨在使用应用程序编程接口处理应用程序基础设施服务之间基于网络的大量进程间通信。服务网格确保了容器化的、通常是临时的应用程序基础设施服务之间的通信是快速、可靠和安全的。网格提供了关键功能，包括服务发现、负载平衡、加密、可观察性、可跟踪性、认证和授权，以及对断路器模式的支持。

#### 了解RPC

我在公司使用过基于Thift协议的RPC框架来完成服务之间的通信，包括使用Thift定义数据结构，包括结构体变量，枚举类型等，使用Thift定义接口字段，在生成的代码中补充接口逻辑。RPC 采用客户机/服务器模式。请求程序就是一个客户机，而服务提供程序就是一个服务器。首先，调用进程发送一个有进程参数的调用信息到服务进程，然后等待应答信息。在服务器端，进程保持睡眠状态直到调用信息的到达为止。当一个调用信息到达，服务器获得进程参数，计算结果，发送答复信息，然后等待下一个调用信息，最后，客户端调用过程接收答复信息，获得进程结果，然后调用执行继续进行。

#### 开源贡献经验

腾讯开源ORM库APIJSON Contributor，完成了 APIJSON 对 ClickHouse 和 Hive 数据库的支持，解决SQL语句和数据库驱动的兼容性，开发了SpringBoot相关的生态项目，获得腾讯开源贡献证书。

百度深度学习平台PaddlePaddle Contributor，为 Paddle Inference 添加了 Java 用户接口。在 Paddle Inference 支持 C++/C 编译的基础上，通过 JNI 实现 Java 代码与 C、C++ 代码交互，解决Java和Native对象管理和同步、内存管理等问题，提供预测部署的本地 Java 开发体验和功能。

#### 3. milestone计划

● 制定实现方案，完善设计文档

与相关人员充分沟通，制定相关实现方案的总体架构，实现细节，包括RPC的实现方案，协议的选择和实现，接入模型设计，服务发现逻辑，服务调用逻辑

● 制定开发计划，完善开发细节

与相关人员充分沟通，评估各个部分的工作量，完善相关实现方案的细节

● 实现基础RPC协议，完成环境控制平面 mock 测试

Envoy 提供的 Java 实例控制平面 https://github.com/envoyproxy/java-control-plane

● 完成 Java 版协议支持

Envoy 提供的 Java 版控制平面 API https://search.maven.org/search?q=g:io.envoyproxy.controlplane
