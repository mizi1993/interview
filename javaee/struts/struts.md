###Spring和Struts的区别？
- strusts：是一种基于MVC模式的一个web层的框架。
- Spring:提供了通用的服务，ioc/di aop,关心的不仅仅web层，应当j2ee整体的一个服务，容器，可以很容易融合不同的技术和开发框架struts hibernate ibatis ejb remote springJDBC springMVC

###struts+spring面试题
1. struts
	 Action是不是线程安全的？如果不是，有什么方式可以保证Action的线程安全？如果是，说明原因
- MVC，分析一下struts是如何实现MVC的
- struts中的几个关键对象的作用(说说几个关键对象的作用)
- spring
	说说AOP和IOC的概念以及在spring中是如何应用的
- Hibernate有哪几种查询数据的方式
- load()和get()的区别
- Struts1 Action是单例模式并且必须是线程安全的，因为仅有Action的一个实例来处理所有的请求。单例策略限制了Struts1 Action能作的事和性能，并且要在开发时特别小心。  
Action资源必须是线程安全的或同步的。Struts2 Action对象为每一个请求产生一个实例，因此没有线程安全问题。（实际上，servlet容器给每个请求产生许多可丢弃的对象，并且不会导致性能和垃圾回收问题）  
- struts是用一组类,servlet 和jsp规范实现mvc的
- ActionFrom ActionServlet Action struts-config.xml
- spring的核心就是IOC,通过指定对象的创建办法,描述对象与服务之间的关系,而不生成对象
- 3种,hql 条件查询() 原生sql
- load()方法认为该数据一定存在,可以放心的使用代理来延时加载 ,如果使用过程中发现了问题,就抛出异常;get()方法一定要获取到真实的数据,否则返回null


###Struts工作机制？为什么要使用Struts？
 工作机制：
 Struts2工作流程：

1. 客户端提交一个HttpServletRequest请求（action或JSP页面）。
- 请求被提交到一系列Filter过滤器，如ActionCleanUp和FilterDispatcher等。
- FilterDispatcher是Struts2控制器的核心，它通常是过滤器链中的最后一个过滤器。
- 请求被发送到FilterDispatcher后，FilterDispatcher询问ActionMapper时候需要调用某个action来处理这个Request。
- 如果ActionMapper决定需要调用某个action，FilterDispatcher则把请求交给ActionProxy进行处理。
- ActionProxy通过Configuration Manager询问框架的配置文件struts.xml，找到调用的action类。
- ActionProxy创建一个ActionInvocation实例，通过代理模式调用Action。
- action执行完毕后，返回一个result字符串，此时再按相反的顺序通过Intercepter拦截器。
- 最后ActionInvocation实例，负责根据struts.xml中配置result元素，找到与之相对应的result，决定进一步输出。


###说下Struts的设计模式
- MVC模式:  
	web应用程序启动时就会加载并初始化ActionServler。用户提交表单时，一个配置好的ActionForm对象被创建，并被填入表单相应的数据，ActionServler根据Struts-config.xml 文件配置好的设置决定是否需要表单验证，如果需要就调用ActionForm的Validate（）验证后选择将请求发送到哪个Action，如果 Action不存在，ActionServlet会先创建这个对象，然后调用Action的execute（）方法。Execute（）从 ActionForm对象中获取数据，完成业务逻辑，返回一个ActionForward对象，ActionServlet再把客户请求转发给 ActionForward对象指定的jsp组件，ActionForward对象指定的jsp生成动态的网页，返回给客户。

- 单例模式
- Factory(工厂模式)：
	定义一个基类===》实现基类方法（子类通过不同的方法）===》定义一个工厂类（生成子类实例）
	===》开发人员调用基类方法

- Proxy(代理模式)


###简单描述Spring framework与Struts的不同之处，整合Spring与Struts有哪些方法，哪种最好，为什么？
Spring是完整的一站式框架，而Struts仅是MVC框架，且着重于MVC中的C。
Spring有三种方式整合Struts：
- 使用 Spring 的 ActionSupport 类整合 Struts；
- 使用 Spring 的 DelegatingRequestProcessor 覆盖 Struts 的 RequestProcessor；
	将 Struts Action 管理委托给 Spring 框架，动作委托最好。（详见使用Spring 更好地处理Struts 动作）
- Spring 2.0新增一种方式：AutowiringRequestProcessor。



###struts默认提供了那些拦截器
- 理解Struts2拦截器
	- Struts2拦截器是在访问某个Action或Action的某个方法，字段之前或之后实施拦截，并且Struts2拦截器是可插拔的，拦截器是ＡＯＰ的一种实现．
	- 拦截器栈（Interceptor Stack）。Struts2拦截器栈就是将拦截器按一定的顺序联结成一条链。在访问被拦截的方法或字段时，Struts2拦截器链中的拦截器就会按其之前定义的顺序被调用。
- 实现Struts2拦截器原理
	Struts2拦截器的实现原理相对简单，当请求struts2的action时，Struts 2会查找配置文件，并根据其配置实例化相对的    拦截器对象，然后串成一个列表，最后一个一个地调用列表中的拦截器

	[链接1](http://www.open-open.com/lib/view/open1349622646010.html)  
	[链接2](http://www.open-open.com/lib/view/open1342570764135.html)


###Struts的validate框架是如何验证的？
>在struts配置文件中配置具体的错误提示，再在FormBean中的validate()方法具体调用。