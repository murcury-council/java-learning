# Tomcat & Jetty 启动

Tomcat & Jetty 在启动的时候会给每个 Web 应用创建一个上下文环境，叫做 ServletContext, 为后面的 Spring 容器提供宿主环境。

Tomcat & Jetty 在启动过程中触发容器初始化事件，Spring 的 ContextLoaderListener 会监听到这个事件，它的 contextInitialized 方法会被调用，在这个方法中，Spring 会初始化全局的 **Spring 根容器**，这个就是 Spring 的 IoC 容器，IoC 容器初始化完毕后，Spring 将其存储到 ServletContext 中，便于以后来获取。

Tomcat & Jetty 在启动过程中还会扫描 Servlet，一个 Web 应用中的 Servlet 可以有多个，以 SpringMVC 中的 DispatcherServlet 为例，这个 Servlet 实际上是一个标准的前端控制器，用以转发、匹配、处理每个 Servlet 请求。

Servlet 一般会延迟加载，当第一个请求到达时，Tomcat & Jetty 发现 DispatcherServlet 还没有被实例化，就调用 DispatcherServlet 的 init 方法，DispatcherServlet 在初始化的时候会建立自己的容器，叫做 **SpringMVC 容器**，用来持有 SpringMVC 相关的 Bean。同时，**SpringMVC 还会通过 ServletContext 拿到 Spring 根容器，并将 Spring 根容器设为 SpringMVC 的父容器**，请注意，SpringMVC 容器可以访问父容器中的 Bean，但是父容器不能访问子容器的 Bean，也就是说 Spring 根容器不能访问 SpringMVC 容器里的 Bean（在 Controller 里可以访问 Service 对象，但在Service 里不可以访问 Controller 对象）。

总结：

* SpringContextLoaderListener 监听到 Tomcat & Jetty 的启动，会初始化 Spring 根容器
* Servlet 是延迟加载的，当有请求到来时才会初始化
* DispatcherServlet 初始化的时候会建立 SpringMVC 容器，并将 Spring 根容器设置为父容器



Servlet 容器，Spring 容器，SpringMVC 容器的关系如下图

<img src="https://img-blog.csdnimg.cn/20190503213508577.jpg"/>

