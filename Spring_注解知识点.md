### Spring中的注解相关的知识点：

#### 1.`@Autowired`基本使用

##### 1. 作用：

`@Autowired`是用来自动装配对象的，默认是通过 类型（`byType`）进行自动装配。此外，`@Autowired`注解的`required`参数默认是true，表示开启自动装配，有些时候我们不想使用自动装配功能，可以将该参数设置成false。

~~~java
// 示例
// TestService1.java
package com.sue.cache.service;

import org.springframework.stereotype.Service;

@Service
public class TestService1 {
    public void test1() {
    }
}

// TestService2.java
package com.sue.cache.service;
import org.springframework.stereotype.Service;

@Service
public class TestService2 {

    @Autowired   // 使用 Autowired进行自动装配，默认使用（byType）进行自动装配
    private TestService1 testService1;

    public void test2() {
    }
}

~~~

​	

#####  2.相同类型的对象不止一个时，如何解决冲突？

~~~java
//示例：
// IUser.java
public interface IUser {
    void say();
}

// 具体的实现者1
@Service
public class User1 implements IUser{
    @Override
    public void say() {
    }
}

// 具体实现者2
@Service
public class User2 implements IUser{
    @Override
    public void say() {
    }
}

// 调用者
@Service
public class UserService {

    @Autowired   // 该处使用 @Autowired 进行自动装配时，就会出现错误，有两个具体的实现者
    private IUser user;
}

~~~

解决方法：

 1. 通过 byType 方式会出现问题，再加上使用 byName 进行装配，只需要在 代码上加上@Qualifier 注解即可

    ~~~java
    
    @Service
    public class UserService {
    
        @Autowired
        @Qualifier("user1") //该注解会根据名字进行匹配
        private IUser user;
    }
    
    /**
    * Qualifier意思是合格者，一般跟Autowired配合使用，需要指定一个bean的名称，通过bean
    * 名称就能找到需要装配的bean。
    */
    
    ~~~

2. 还可以通过 `@Primary`注解解决上面的问题。在User1上面加上@Primary注解，表示当出现多个实例，制定的这个为主要的

   ~~~java
   @Primary
   @Service
   public class User1 implements IUser{
       @Override
       public void say() {
       }
   }
   ~~~

   

#### 2. @Autowired的使用范围

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811095237333.png" alt="image-20210811095237333" style="zoom:80%;" />



#### 3.  @Autowired的高端玩法

其实上面举的例子都是通过@Autowired自动装配单个实例，但这里我会告诉你，它也能自动装配多个实例，怎么回事呢？

将UserService方法调整一下，用一个List集合接收IUser类型的参数：

~~~java
/**
 * 实现一个具体的接口
 */
public interface Iuser {
    void say();
}

// 实现注入,第一个实现类
@Service
public class User1 implements Iuser{
    @Override
    public void say() {
        System.out.println("user1");
    }
}

// 实现注入，第二个实现类
@Service
public class User2 implements Iuser{
    @Override
    public void say() {
        System.out.println("user2");
    }
}

/**
 * 进行测试通过 @Autowired 装配多个实例
 */
@Service
public class UserService {

    @Autowired  // 注入的是一个 list
    private List<Iuser> iuserList;

    @Autowired  // 注入一个   Set
    private Set<Iuser> userSet;

    @Autowired  // 注入的一个 Map
    private Map<String, Iuser> userMap;

    public void test(){
        System.out.println("list: " + iuserList);
        System.out.println("set: " + userSet);
        System.out.println("map: " + userMap);
    }
}

/**  输出结果，将注入的两个实现类都注入进 Spring中
* list:[com.redisspringboot.testServer.User1@c3177d5,com.redisspringboot.testServer.User2@76f856a8]
set: [com.redisspringboot.testServer.User1@c3177d5,com.redisspringboot.testServer.User2@76f856a8]
map: {user1=com.redisspringboot.testServer.User1@c3177d5,user2=com.redisspringboot.testServer.User2@76f856a8}
*
*/
~~~



#### 4. @Autowired一定能装配成功？

有些情况下，即使使用了@Autowired装配的对象还是null，到底是什么原因呢？

	1. 没有加 @Controller、@Service、@Component、@Repository等注解，spring就无法完成自动装配的功能
 	2. 注入 Filter 或者 Listener中，使用到在 某一个容器中的对象，但是springmvc的启动是在DisptachServlet里面做的，而它是在listener和filter之后执行。如果我们想在listener和filter里面@Autowired某个bean，肯定是不行的，因为filter初始化的时候，此时bean还没有初始化，无法自动装配。

~~~java
// 实现的 Filter 过滤器
@Slf4j
public class MyFilter implements Filter {

    // 注入一个 bean 对象
    @Autowired      // Filter的启动是在 bean初始化之前，这个时候注入对象会出错
    private Iuser iuser;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

        iuser.say();

        log.info("过滤器开始************");
        System.out.println("进行过滤以及逻辑判断操作");
        filterChain.doFilter(servletRequest,servletResponse);
    }
}

~~~



~~~java
// 将过滤器注入到 Spring 中
@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean filterRegistrationBean(){

        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.setFilter(new MyFilter());
        bean.addUrlPatterns("/*");  // 配置过滤内容
        
        return bean;
    }
}
~~~

通过上述方式注入过滤器，会出现错误，

![image-20210811130149287](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811130149287.png)



解决方法1：

答案是使用 WebApplicationContextUtils.getWebApplicationContext 获取当前的 ApplicationContext ，再通过它获取到bean实例。

~~~java
@Slf4j
public class MyFilter implements Filter {

    // 注入一个 bean 对象
    private Iuser iuser;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        ApplicationContext applicationContext = WebApplicationContextUtils.getWebApplicationContext(filterConfig.getServletContext());
        this.iuser = ((Iuser)(applicationContext.getBean("user1")));
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        
        iuser.say();

        log.info("过滤器开始************");
        System.out.println("进行过滤以及逻辑判断操作");
        filterChain.doFilter(servletRequest,servletResponse);
    }
}

~~~

![image-20210811130721802](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811130721802.png)

​	

第二种解决办法：

```java
/**
使用@ServletComponentScan注解的含义在SpringBootApplication(启动类)上使用@ServletComponentScan注解后，Servlet、Filter(过滤器)、Listener(监听器)可以直接通过@WebServlet、@WebFilter、@WebListener注解自动注册，无需其他代码！，
不需要写 FilterRegistrationBean
*/
@Configuration
public class FilterConfig {
    @Bean
    public FilterRegistrationBean filterRegistrationBean(){

        FilterRegistrationBean bean = new FilterRegistrationBean();
        bean.setFilter(new MyFilter());
        bean.addUrlPatterns("/*");

        return bean;
    }
}

```



~~~java
@Slf4j
@WebFilter(urlPatterns = "/*")   // 开启注解扫描
public class MyFilter implements Filter {

    // 注入一个 bean 对象 
    @Autowired   // 自动注入
    private Iuser iuser;

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

        iuser.say();
        log.info("过滤器开始************");
        System.out.println("进行过滤以及逻辑判断操作");
        filterChain.doFilter(servletRequest,servletResponse);
    }
}
~~~



~~~java
// 入口方法上面开启扫描
@SpringBootApplication
@ServletComponentScan   // 进行扫描 servlet Filter Listener等
public class RedisSpringbootApplication extends SpringBootServletInitializer {
    
    public static void main(String[] args) {
        SpringApplication.run(RedisSpringbootApplication.class, args);
    }
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(RedisSpringbootApplication.class);
    }
}
~~~

![image-20210811131434016](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811131434016.png)



#### 5. @Autowired和@Resouce的区别

@Autowired功能虽说非常强大，但是也有些不足之处。比如：比如它跟spring强耦合了，如果换成了JFinal等其他框架，功能就会失效。而@Resource是JSR-250提供的，它是Java标准，绝大部分框架都支持。

我们重点看看@Autowired和@Resource的区别。

- @Autowired默认按byType自动装配，而@Resource默认byName自动装配。
- @Autowired只包含一个参数：required，表示是否开启自动准入，默认是true。而@Resource包含七个参数，其中最重要的两个参数是：name 和 type。
- @Autowired如果要使用byName，需要使用@Qualifier一起配合。而@Resource如果指定了name，则用byName自动装配，如果指定了type，则用byType自动装配。
- @Autowired能够用在：构造器、方法、参数、成员变量和注解上，而@Resource能用在：类、成员变量和方法上。
- @Autowired是spring定义的注解，而@Resource是JSR-250定义的注解。
- 装配顺序不同

<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811131931999.png" alt="image-20210811131931999" style="zoom:80%;" />



<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811131957120.png" alt="image-20210811131957120" style="zoom:80%;" />



<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811132051527.png" alt="image-20210811132051527" style="zoom:80%;" />



<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811132103272.png" alt="image-20210811132103272" style="zoom:80%;" />



<img src="C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210811132027918.png" alt="image-20210811132027918" style="zoom:80%;" />



### SpringBoot 中的监听器

1. ###### 监听器介绍

web 监听器是一种 Servlet 中特殊的类，它们能帮助开发者监听 web 中特定的事件，比如 ServletContext, HttpSession, ServletRequest 的创建和销毁；变量的创建、销毁和修改等。可以在某些动作前后增加处理，实现监控。

2. ###### `SpringBoot`中的监听器 

参考：https://www.jianshu.com/p/e89fe42ea1bb

写一个监听器，实现 `ApplicationListener<ContextRefreshedEvent>` 接口，重写 `onApplicationEvent` 方法

3. ###### 监听器的使用场景：

1. 监听 `servlet `上下文用来初始化一些数据， 可以用于缓存，实现接口 (`ApplicationListener`)
2. 监听 http session 用来获取当前在线的人数, 实现接口(`HttpSessionListener `)
3. 监听客户端请求的 servlet request 对象来获取用户的访问信息,实现 （`ServletRequestListener `）



##### 在Springboot中自定义监听事件

自定义监听器的实现步骤：

~~~
第一步：自定义事件。创建自定义事件类，继承 ApplicationEvent 对象

第二步：自定义监听器。创建自定义监听器类监听自定义事件，需要实现ApplicationListener 接口，重写onApplicationEvent 方法

第三步：设定事件触发类，使用 applicationContext.publishEvent(event)手动触发监听事件

~~~





![img](https://img-blog.csdnimg.cn/20200702165450664.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0MjA3NDIy,size_16,color_FFFFFF,t_70)



监听器(listener)：就是对项目起到监听的作用，它能感知到包括request(请求域)，session(会话域)和applicaiton(应用程序)的初始化和属性的变化；

过滤器(filter)：就是对请求起到过滤的作用，它在监听器之后，作用在servlet之前，对请求进行过滤；

拦截器(interceptor)：就是对请求和返回进行拦截，它作用在servlet的内部，具体来说有三个地方：

1）servlet_1和servlet_2之间，即请求还没有到controller层

2）servlet_2和servlet_3之间，即请求走出controller层次，还没有到渲染时图层

3）servlet_3和servlet_4之间，即结束视图渲染，但是还没有到servlet的结束



##### SpringBoot中 使用 过滤器、监听器、拦截器

1. ###### 过滤器：

过滤器 Filter，是Servlet中用于对请求进行拦截，一般可用于 读取 Session 判断用户是否登录，判断请求的URL的权限，可以对请求预处理，解决文件编码问题

在SpringBoot中使用过滤器：

​	1. `@WebFilter`时`Servlet3.0`新增的注解，原先实现过滤器，需要在`web.xml`中进行配置，而现在通过此注解，启动时会自动扫描自动注册。

2. 实现 Filter接口，实现其中的 `doFilter` 方法
3. 启动类加入`@ServletComponentScan`注解即可





2. ###### 拦截器

1.  **编写自定义拦截器类**，主要是实现 `HandlerInterceptor`接口，实现其中方法 `preHandle`, `postHandle`, `afterCompletion`
2. **通过继承`WebMvcConfigurerAdapter`注册拦截器**



请求链路说明：

![img](https://upload-images.jianshu.io/upload_images/13169342-8c2f2b7fa33ba431.png?imageMogr2/auto-orient/strip|imageView2/2/w/819/format/webp)











































































































































































