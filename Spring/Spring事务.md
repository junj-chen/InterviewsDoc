

Spring中事务失效的场景：

![img](https://mmbiz.qpic.cn/mmbiz_jpg/uL371281oDFI5ibhP1TXOMnqQtJhfb3XCnTbgmpiab2LDA8VVCmg2jMUoeJd70gAJsj7vL2IB0icYxsbsvnKIu9LQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



事务不生效：

	1. 访问权限问题

java的访问权限主要有四种：private、default、protected、public，它们的权限从左到右，依次变大。把有某些事务方法，定义了错误的访问权限，就会导致事务功能出问题，例如：

~~~java
@Service
public class UserService {
    
    @Transactional  // 定义的 private的方法，导致事务失效
    private void add(UserModel userModel) {
         saveData(userModel);
         updateData(userModel);
    }
}
~~~

我们可以看到add方法的访问权限被定义成了`private`，这样会导致事务失效，spring要求被代理方法必须是`public`的。

说白了，在`AbstractFallbackTransactionAttributeSource`类的`computeTransactionAttribute`方法中有个判断，如果目标方法不是public，则`TransactionAttribute`返回null，即不支持事务。

也就是说，如果我们自定义的事务方法（即目标方法），它的访问权限不是`public`，而是private、default或protected的话，spring则不会提供事务功能。



2. 方法使用 final 或者 static修饰、

有时候，某个方法不想被子类重新，这时可以将该方法定义成final的。普通方法这样定义是没问题的，但如果将事务方法定义成final，例如：

~~~java
@Service
public class UserService {

    @Transactional   // final修饰的方法，事务不生效
    public final void add(UserModel userModel){
        saveData(userModel);
        updateData(userModel);
    }
}
~~~

如果你看过spring事务的源码，可能会知道spring事务底层使用了aop，也就是通过jdk动态代理或者cglib，帮我们生成了代理类，在代理类中实现的事务功能。

但如果某个方法用final修饰了，那么在它的代理类中，就无法重写该方法，而添加事务功能。



3. 方法内部调用

有时候我们需要在某个Service类的某个方法中，调用另外一个事务方法，比如：

~~~java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Transactional
    public void add(UserModel userModel) {
        userMapper.insertUser(userModel);  // 该方法执行事务是因为底层生成了代理对象
        updateStatus(userModel);  // 该方法直接调用，没有使用到代理对象，所以无法执行事务
    }

    @Transactional
    public void updateStatus(UserModel userModel) {
        doSameThing();
    }
}
~~~

我们看到在事务方法add中，直接调用事务方法updateStatus。从前面介绍的内容可以知道，updateStatus方法拥有事务的能力是因为spring aop生成代理了对象，但是这种方法直接调用了this对象的方法，所以updateStatus方法不会生成事务。

由此可见，在同一个类中的方法直接内部调用，会导致事务失效。

解决方法：

​	3.1  在该Service类中注入自己

如果不想再新加一个Service类，在该Service类中注入自己也是一种选择。具体代码如下：

~~~java
@Servcie
public class ServiceA {
   @Autowired
   prvate ServiceA serviceA;  // 注入自身，实现事务

   public void save(User user) {
         queryData1();
         queryData2();
         serviceA.doSave(user);
   }

   @Transactional(rollbackFor=Exception.class)
   public void doSave(User user) {
       addData1();
       updateData2();
    }
 }
~~~



4. 未被spring管理

即使用spring事务的前提是：对象要被spring管理，需要创建bean实例。通常情况下，我们通过@Controller、@Service、@Component、@Repository等注解，可以自动实现bean实例化和依赖注入的功能。



5. 表不支持事务

在mysql5之前，默认的数据库引擎是`myisam`。它的好处就不用多说了：索引文件和数据文件是分开存储的，对于查多写少的单表操作，性能比innodb更好。有些老项目中，可能还在用它。在创建表的时候，只需要把`ENGINE`参数设置成`MyISAM`即可：

myisam好用，但有个很致命的问题是：`不支持事务`。



二 事务不回滚

1. 错误的传播特性

使用`@Transactional`注解时，是可以指定`propagation`参数的。

该参数的作用是指定事务的传播特性，spring目前支持7种传播特性：

- `REQUIRED` 如果当前上下文中存在事务，那么加入该事务，如果不存在事务，创建一个事务，这是默认的传播属性值。
- `SUPPORTS` 如果当前上下文存在事务，则支持事务加入事务，如果不存在事务，则使用非事务的方式执行。
- `MANDATORY` 如果当前上下文中存在事务，否则抛出异常。
- `REQUIRES_NEW` 每次都会新建一个事务，并且同时将上下文中的事务挂起，执行当前新建事务完成以后，上下文事务恢复再执行。
- `NOT_SUPPORTED` 如果当前上下文中存在事务，则挂起当前事务，然后新的方法在没有事务的环境中执行。
- `NEVER` 如果当前上下文中存在事务，则抛出异常，否则在无事务环境上执行代码。
- `NESTED` 如果当前上下文中存在事务，则嵌套事务执行，如果不存在事务，则新建事务。

~~~java
@Service
public class UserService {
    
    @Transactional(propagation = Propagation.NEVER)
    public void add(UserModel userModel) {
        saveData(userModel);
        updateData(userModel);
    }
}
~~~



可以看到add方法的事务传播特性定义成了Propagation.NEVER，这种类型的传播特性不支持事务，如果有事务则会抛异常。

2. 自己吞了异常

事务不会回滚，最常见的问题是：开发者在代码中手动try...catch了异常。比如：

~~~java
@Slf4j
@Service
public class UserService {
    
    @Transactional
    public void add(UserModel userModel) {
        try {
            saveData(userModel);
            updateData(userModel);
        } catch (Exception e) {
            log.error(e.getMessage(), e);
        }
    }
}
~~~

spring事务当然不会回滚，因为开发者自己捕获了异常，又没有手动抛出，换句话说就是把异常吞掉了。如果想要spring事务能够正常回滚，必须抛出它能够处理的异常。如果没有抛异常，则spring认为程序是正常的。



事务中的推荐方法：

上面聊的这些内容都是基于`@Transactional`注解的，主要说的是它的事务问题，我们把这种事务叫做：`声明式事务`。

spring还提供了另外一种创建事务的方式，即通过手动编写代码实现的事务，我们把这种事务叫做：`编程式事务`。例如：

~~~java
 @Autowired
   private TransactionTemplate transactionTemplate;  // 编程式事务
   
   
   public void save(final User user) {
         queryData1();
         queryData2();
         transactionTemplate.execute((status) => {  // 执行代码块
            addData1();
            updateData2();
            return Boolean.TRUE;
         })
   }
~~~

在spring中为了支持编程式事务，专门提供了一个类：TransactionTemplate，在它的execute方法中，就实现了事务的功能。

相较于`@Transactional`注解声明式事务，我更建议大家使用，基于`TransactionTemplate`的编程式事务。主要原因如下：

1. 避免由于spring aop问题，导致事务失效的问题。
2. 能够更小粒度的控制事务的范围，更直观。