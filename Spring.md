1. ##### Spring 是什么？

   Spring 是一个轻量级的IOC和AOP容器，是 java 应用程序提供基础性服务的一套框架，目的是用于简化企业应用程序的开发。使得开发者只需要关心业务的需求。

   包含以下7个模块：

   1. Spring Context：提供框架式的Bean访问方式，以及企业级的功能
   2. Spring Core: 核心类库，所用的功能依赖于该类 提供 IOC 和 DI服务
   3. Spring AOP：AOP服务
   4. Spring Web：提供了基本的面向web的综合特性
   5. Spring MVC：提供面向Web应用的Model - view-Controller
   6. Spring DAO：对JDBC的抽象封装，简化了数据访问异常的处理，统一管理JDBC
   7. Spring ORM：对现有的ORM框架的支持



2. ##### Spring 的优点？

   1. Spring属于低侵入式设计，代码污染低
   2. Spring的DI机制将对象之间的依赖关系交给框架处理，降低组件的耦合性
   3. Spring提供了AOP技术，支持将一些通用的任务（事务、安全、权限）进行集中式管理，从而提供更好的复用
   4. Spring对于主流的应用框架提供了集成支持

3. #####  Spring IOC理解

   1. IOC就是控制反转，指创建对象的控制权转移给Spring框架进行管理，并且可以根据配置文件 创建和管理每个实例之间的依赖关系，对象与对象之间松耦合，利用功能的复用。
   2. Spring的IOC有三种注入方式，构造器注入、setter方法注入、根据注解注入

   

4. #####  Spring AOP的理解

   1. AOP是面向切面，也就是将一些与业务无关，具有公共行为和逻辑的内容封装成一个可重用的模块，减少系统中重复的代码块，降低模块之间的耦合度，提高系统的可维护性

   2. AOP的实现关键在于代理模式，代理模式可以分为静态代理和动态代理，Spring AOP中使用动态代理

   动态代理：

   ​	1. 所谓的动态代理就是说AOP框架不会去修改字节码，而是每次运行时在内存中临时为方法生成一个AOP对象，这个AOP对象包含了目标对象的全部方法，并且在特定的切点做了增强处理，并回调原对象的方法。




















































































































































































































