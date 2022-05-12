### Apache Dubbo GSoC2022 Proxyless Mesh support

#### 1. Scheme design

#### Objective

Directly implement xDS protocol analysis through Dubbo3 SDK of different language versions, realize direct communication between Dubbo and Control Plane, and then realize unified control of control face traffic control, service governance, observability, security, etc., and avoid the performance brought by Sidecar mode Attrition and deployment architecture complexity.

#### Proposal

[xds-2](https://github.com/apache/dubbo-awesome/blob/master/images/mesh-xds-2.png)

In the Dubbo3 Proxyless architecture mode, the Dubbo process will communicate directly with the control plane, and the Dubbo process will continue to maintain the direct communication mode. We can see the advantages of the Proxyless architecture:

- No additional proxy loss, so it is more suitable for performance sensitive applications
- More conducive to smooth migration of legacy systems
- Simple architecture, easy to operate and deploy
- Works with almost any deployment environment

#### Architecture

[xds-1](../images/mesh-xds-1.png)

#### Details

1. Official description of xDS

https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol

● Listener Discovery Service (LDS): Returns Listener resources. Used basically as a convenient root for the gRPC client's configuration. Points to the RouteConfiguration.
● Route Discovery Service (RDS): Returns RouteConfiguration resources. Provides data used to populate the gRPC service config. Points to the Cluster.
● Cluster Discovery Service (CDS): Returns Cluster resources. Configures things like load balancing policy and load reporting. Points to the ClusterLoadAssignment.
● Endpoint Discovery Service (EDS): Returns ClusterLoadAssignment resources. Configures the set of endpoints (backend servers) to load balance across and may tell the client to drop requests.

2. Protocol base: protobuf + gRPC

Implementation 1: Manually create native gRPC services using io.grpc dependencies
Implementation 2: Using Dubbo 3.0's New Protocol

3. Test environment control plane mock

Java instance control plane provided by Envoy https://github.com/envoyproxy/java-control-plane

4. Java Edition Protocol Support

Java version control plane API provided by Envoy https://search.maven.org/search?q=g:io.envoyproxy.controlplane

Agreement Available Content

● LDS: envoy.api.v2.Listener
● RDS: envoy.api.v2.RouteConfiguration
● CDS: envoy.api.v2.Cluster
● EDS: envoy.api.v2.ClusterLoadAssignment

LDS

Listener discovery protocol. In sidecar deployment mode, users configure local listeners. Since Dubbo access to xDS itself is a proxy-less mode, many configurations of LDS do not need to be considered.
For the LDS protocol, a listener that needs to be adapted is used to obtain the routing configuration.
For the LDS protocol, some control planes cannot guarantee that the acquired listener is unique, so it needs to be compatible with the configuration when multiple listeners are acquired.

RDS

Route discovery protocol. Through route configuration, the requested address can be matched, and the corresponding upstream Cluster can be obtained after a certain strategy.
For the RDS protocol, you can obtain a set of corresponding routing protocols based on the configuration of the Listener. Then filter out the corresponding Cluster according to the service name corresponding to the service introspection framework.
For the RDS protocol, some control planes cannot guarantee that the obtained routing group is unique, so it needs to be compatible with the configuration when multiple routing groups are obtained.
The flow control and retry configurations are included in the RDS results, which can be used as the default configuration source for the routing configuration and load balancing sections.

CDS

Cluster discovery protocol, which can obtain the configuration in the corresponding cluster (such as timeout time, retry limit, metadata information, etc.), is obtained based on RDS.
The results of the CDS can be used as a configuration source for the routing configuration and load balancing sections.

EDS

The Endpoint discovery protocol can obtain the corresponding node information in the cluster, that is, the underlying node information of inter-service communication.
Through the results of EDS, the corresponding node address information, weight information, and health information can be obtained, and then constitute the most basic instance information of the service introspection framework.

Access model design

The mode of the registry is used for docking, and the configuration is passed out through the ServiceInstance runtime configuration, and passed to the service invocation through the Invoker.

service discovery logic

The xDS access is connected in the mode of the registry. The node discovery is the same as the service introspection model of other registries. The load balancing and routing configuration of xDS is passed out through the dynamic runtime configuration of ServiceInstance, and the configuration parameters are passed in when building the Invoker. Configure the address.


service call logic

In the RPC call link, when the load balancing strategy is obtained, the load balancing information (weight information adapted to the EDS) is constructed according to the parameter information of the Invoker, and the configuration of the tool RDS and CDS is modified for the retry logic configuration of the cluster.

#### 2. Personal Skill Description

I am a student at Beijing University of Posts and Telecommunications，have plenty of time after school and during holidays. I am very interested in learning and exploring different areas of computing, and I have been fortunate enough to participate in the open source community and be a contributor to several projects over the past period. I hope to learn more about the open source community and contribute to Dubbo. I have solid computer and networking fundamentals, familiar with java programming, and understand Service Mesh and RPC. I like to learn knowledge, analyze problems, hope to grow with the Dubbo community, and be willing to spend enough time and energy.

https://github.com/chenyanlann

#### Programming language

Familiar with JAVA programming, including Java foundation, Java container, Java concurrency, JVM, JNI, etc., and have read books such as "Java Core Technology", "Effective Java Chinese Edition", "In-depth Understanding of Java Virtual Machine" and so on, while Dubbo is a distributed framework developed in JAVA.

#### Basic computer knowledge

Understand computer basics such as network protocols, operating systems, data structures, etc. The main courses of the school include programming practice, computer network, data structure and algorithm, etc. RPC is a method of requesting services from remote computer programs through the network, based on the underlying network transmission protocol, such as TCP or UDP, to carry information between communication programs. data. In the OSI network communication model, RPC spans the transport layer and the application layer. RPC makes it easier to develop applications including network distributed multiprogramming.

#### Understanding Service Mesh

I have used Mesh services in my company to complete the communication between services, including outgoing traffic Mesh and ingress traffic Mesh, through which Mesh realizes the diversion and direction of request traffic. A service mesh is a configurable, low-latency infrastructure layer designed to handle heavy network-based interprocess communication between application infrastructure services using application programming interfaces. A service mesh ensures that communication between containerized, often ephemeral, application infrastructure services is fast, reliable, and secure. The grid provides key capabilities including service discovery, load balancing, encryption, observability, traceability, authentication and authorization, and support for the circuit breaker pattern.

#### Understanding RPC

I have used the RPC framework based on the Thift protocol in my company to complete the communication between services, including using Thift to define data structures, including structure variables, enumeration types, etc., using Thift to define interface fields, and supplementing interface logic in the generated code . RPC uses a client/server model. The requestor is a client, and the service provider is a server. First, the calling process sends a call message with process parameters to the server process, and then waits for a reply message. On the server side, the process stays asleep until the call information arrives. When a call message arrives, the server obtains the process parameters, calculates the result, sends a reply message, and then waits for the next call message. Finally, the client calls the process to receive the reply message, obtain the process result, and then call execution to continue.

#### Open source contribution experience

I am the APIJSON Contributor of Tencent's open source ORM library. I have completed APIJSON's support for ClickHouse and Hive databases, solved the compatibility of SQL statements and database drivers, developed SpringBoot-related ecological projects, and obtained the Tencent Open Source Contribution Certificate.

I'm the PaddlePaddle Contributor of Baidu's deep learning platform, adding a Java user interface to Paddle Inference. On the basis that Paddle Inference supports C++/C compilation, it realizes the interaction between Java code and C and C++ code through JNI, solves problems such as Java and Native object management and synchronization, memory management, etc., and provides local Java development experience and functions for predictive deployment.

#### 3. milestone plan

● Develop implementation plans and improve design documents

Fully communicate with relevant personnel to formulate the overall architecture and implementation details of the relevant implementation plan, including RPC implementation plan, protocol selection and implementation, access model design, service discovery logic, service invocation logic

● Make a development plan and improve the development details

Fully communicate with relevant personnel, evaluate the workload of each part, and improve the details of the relevant implementation plan

● Implement the basic RPC protocol and complete the mock test of the environment control plane

Java instance control plane provided by Envoy https://github.com/envoyproxy/java-control-plane

● Complete Java Edition protocol support

Java version control plane API provided by Envoy https://search.maven.org/search?q=g:io.envoyproxy.controlplane
