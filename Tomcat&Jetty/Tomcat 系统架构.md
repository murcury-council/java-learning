# Tomcat 系统架构

Tomcat 需要实现 2 个核心功能：

* 处理 Socket 连接，负责网络字节流与 Request 和 Response 对象的转化
* 加载和管理 Servlet，以及具体处理 Request 请求

因此 Tomcat 设计了两个核心组件，连接器和容器来分别做这两件事情。连接器负责对外交流，容器负责内部处理。

Tomcat 为了实现支持多种 I/O 和应用层协议，一个容器可能对接多个连接器。单独的连接器和容器都不能对外提供服务，需要组装起来工作，组装后的整体叫作 Service 组件。Tomcat 内可能有多个 Service。通过在 Tomcat 中配置多个 Service，可以实现通过不同的端口来访问同一台机器上部署的不同应用。

<img src="https://static001.geekbang.org/resource/image/ee/d6/ee880033c5ae38125fa91fb3c4f8cad6.jpg" style="zoom:40%;" />

## 连接器

#### 作用

使 I/O 和应用层协议对容器透明

* 网络通信
* 应用层协议解析
* Tomcat Request / Response 与 ServletRequest/ServletResponse 的转化

为了实现上述功能，Tomcat 设计者分别设计了 Endpoint 、Processor、Adapter 三个组件来实现上述三个功能。

* Endpoint 负责提供字节流给 Processor (网络通信)
* Processor 负责提供 Tomcat Request 对象给 Adapter (应用层协议解析)
* Adapter 负责提供 ServletRequest 给容器 (对象转化)

由于 I/O 模型可以和应用层协议自由组合，所以讲网络通信和协议解析放在了一起考虑，设计了 ProtocolHandler 接口来封装这两种变化点。各种协议和通信模型的组合对接口进行具体实现。

<img src="https://static001.geekbang.org/resource/image/6e/ce/6eeaeb93839adcb4e76c15ee93f545ce.jpg" style="zoom:40%;"/>

#### ProtocolHandler 组件

##### Endpoint

通信端点，即通信监听的接口，是具体的 Socket 接收和发送处理器，用来实现 TCP/IP 协议。有两个重要的子组件：

* Acceptor

  用于监听 Socket 连接请求

* SocketProcessor

  用于处理收到的 Socket 请求，实现 Runnable 接口，在 run 方法里调用协议处理组件 Processor 进行处理。为了提高处理能力，SocketProcessor 被提交到线程池来执行。而这个线程池叫作执行器（Executor）

##### Processor

对应用层协议的抽象

<img src="https://static001.geekbang.org/resource/image/30/cf/309cae2e132210489d327cf55b284dcf.jpg" style="zoom:40%;" />

#### Adapter 组件

##### CoyoteAdapter

Tomcat Request 转化成 ServletRequest