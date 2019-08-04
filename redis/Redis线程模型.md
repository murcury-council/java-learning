# Redis线程模型

Redis基于Reactor模式开发了自己的网络事件处理器：文件事件处理器（File Event Handler）

* 文件事件处理器使用I/O多路复用程序来同时监听多个Socket，并根据Socket目前执行的任务来为Socket关联不同的事件处理器

<img src="https://img.draveness.me/2016-11-26-redis-reactor-pattern.png-1000width" width="700"/>

#### 文件事件处理器类型

Redis有大量的事件处理器类型，一个简单命令主要涉及如下三种事件处理器：

* acceptTcpHandler 连接应答处理器，负责处理客户端连接相关的事件，当有client连接到Redis时会产生AE_READABLE事件，触发此Handler
* readQueryFromClient 命令请求处理器，负责读取客户端通过Socket发送的命令
* sendReplyToClient 命令回复处理器，当Redis的处理完命令，数据准备完毕，就会产生AE_WRITEABLE事件



###### 客户端和Redis的一次通信流程

redis初始化时，会将连接应答处理器和AE_READABLE事件关联起来

当client连接到redis时，会产生一个AE_READABLE事件，触发连接应答处理器，处理器对请求进行应答，创建客户端Socket，并将AE_READABLE事件和命令请求处理器进行关联

之后，client向redis发送命令，产生AE_READABLE事件，触发命令请求处理器，处理器读取命令内容，进行执行和处理。

当redis准备好数据后，就会将Socket的AE_WRITEABLE事件和命令回复处理器关联，当client准备好读取响应数据时，会产生AE_WRITEABLE事件，触发命令回复处理器，处理器将响应数据写入Socket给client

命令回复处理器写完后，会删除这个Socket的AE_WRITEABLE事件和命令回复处理器的关联关系

#### I/O多路复用

<img src="https://img.draveness.me/2016-11-26-I:O-Multiplexing-Model.png-1000width" width="700"/>

文件描述符（File Descriptor）

`select/epoll` 监听多个FD，当FD变得可读或可写时将其加入到`eventLoop`的`fired`数组中

`select`最多只能监听1024个FD，当调用select后阻塞，当有FD就绪时，select返回，需要遍历才能知道哪些FD就绪

`epoll`没有监听FD数量的限制，当FD就绪时，`epoll_wait`返回，返回时会提供一个`epoll_event`数组，保存了发生的`epoll`事件以及发生该事件的FD，直接将这个FD加入到`eventLoop`的`fired`数组中即可，不需要遍历