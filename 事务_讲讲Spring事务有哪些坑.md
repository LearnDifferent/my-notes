由于[《面试官:谈谈你对mysql事务的认识》](http://mp.weixin.qq.com/s?__biz=MzIwMDgzMjc3NA==&mid=2247484867&idx=1&sn=7c781f148f566fc3ee98676bcb61fe12&chksm=96f667aaa181eebcdf75432bb367efc9e5af0752cc2a81c1093e2dcf73f2193d80666d91fc13&scene=21#wechat_redirect)篇幅所限，因此略过了spring事务相关常见面试题，今天给大家补上！主要题目如下:

- (1)spring事务的原理?
- (2)spring什么情况下进行事务回滚？
- (3)spring事务什么时候失效?
- (4)Spring的事务和数据库的事务隔离是一个概念么？
- (5)spring事务控制放在service层，在service方法中一个方法调用service中的另一个方法，默认开启几个事务？
- (6)怎么保证spring事务内的连接唯一性?

## **正文**

**1、spring事务的原理?** 首先，我们先明白spring事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，spring是无法提供事务功能的。 那么，我们一般使用JDBC操作事务的时候，代码如下

- (1)获取连接 Connection con = DriverManager.getConnection()
- (2)开启事务con.setAutoCommit(true/false);
- (3)执行CRUD
- (4)提交事务/回滚事务 con.commit() / con.rollback();
- (5)关闭连接 conn.close();

使用spring事务管理后，我们可以省略步骤(2)和步骤(4)，就是让AOP帮你去做这些工作。关键类在TransactionAspectSupport这个切面里，大家有兴趣自己去翻。我就不列举了，因为公众号类型的文章，实在不适合写一些源码解析！

**2、spring 什么情况下进行事务回滚？** 首先，我们要明白Spring事务回滚机制是这样的：当所拦截的方法有指定异常抛出，事务才会自动进行回滚！ 因此，如果你默默的吞掉异常，像下面这样

```java
@Service
public class UserService{
    @Transactional
    public void updateUser(User user) {
        try {
            System.out.println("孤独烟真帅");
            //do something
        } catch {
          //do something
        }
    }

}
```

那切面捕捉不到异常，肯定是不会回滚的。 还有就是，默认配置下，事务只会对Error与RuntimeException及其子类这些异常，做出回滚。一般的Exception这些Checked异常不会发生回滚（如果一般Exception想回滚要做出配置），如下所示

```java
@Transactional(rollbackFor = Exception.class)
```

但是在实际开发中，我们会遇到这么一种情况！就是并没有异常发生，但是由于事务结果未满足具体业务需求，所以我们需要手动回滚事务，于是乎方法也很简单

- (1)自己在代码里抛出一个自定义异常(常用)
- (2)通过编程代码回滚(不常用)

```java
TransactionAspectSupport.currentTransactionStatus()
.setRollbackOnly();
```

**3、spring事务什么时候失效?** `ps:`经典老题啊！！4年前我毕业那会在问，我都工作4年了，现在还问这道！其出现频率，不下于HashMap的出现频率！该问题有很多问法，例如spring事务有哪些坑?你用spring事务的时候，有遇到过什么问题么？其实答案都一样的，OK，不罗嗦了，开始答案！

我们知道spring事务的原理是AOP，进行了切面增强，那么失效的根本原因是这个AOP不起作用了！常见情况有如下几种 *(1)发生自调用* 例如代码如下

```javascript
@Service
public class UserService{
   public void update(User user) {
        updateUser(user);
    }

    @Transactional
    public void updateUser(User user) {
        System.out.println("孤独烟真帅");
        //do something
    }

}
```

此时是无效的，因此上面的代码等同于

```java
@Service
public class UserService{
   public void update(User user) {
        this.updateUser(user);
    }

    @Transactional
    public void updateUser(User user) {
        System.out.println("孤独烟真帅");
        //do something
    }

}
```

此时这个this对象不是代理类，而是UserService对象本身！ 解决方法很简单，让那个this变成UserService的代理类即可，就不展开说明了！

*(2)方法不是public的* OK，我这里不想举源码。大家想一个逻辑就行！ @Transactional注解的方法都是被外部其他类调用才有效！ 如果方法修饰符是private的，这个方法能被外部其他类调到么？ 既然调不到，事务生效有意义么？ 想通这套逻辑就行了~~

记住:@Transactional 注解只能应用到 public 可见度的方法上。如果你在 protected、private 或者 package-visible 的方法上使用 @Transactional 注解，它也不会报错， 但是这个被注解的方法将不会有事务行为。

`ps`:先这么理解就好了，因为真的去翻原因，就要贴代码了，这文章可读性就很差了。

*(3)发生了错误异常* 这个问题在第二问讲过了，因为默认回滚的是：RuntimeException。如果是其他异常想要回滚，需要在@Transactional注解上加rollbackFor属性。 又或者是异常被吞了，事务也会失效，不赘述！

*(4)数据库不支持事务* 毕竟spring事务用的是数据库的事务，如果数据库不支持事务，那spring事务肯定是无法生效滴！

OK，答到这里就够了！

可能有的读者会说了

> **烟哥啊，其他文章里说什么数据源没有配置事务管理器也会导致事务失效，你怎么没提？**

OK，我为什么不提，因为这种情况属于你配置的不对！随便少一个配置都会导致事务不生效，例如我们在Springboot中的Application类上不加注解`@EnableTransactionManagement`，也会使事务不生效，难道您能将每种情况下的配置背下来？这种配置的东西，临时查询即可！再比如，你把隔离级别配置成

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

该隔离级别表示不以事务运行，当前若存在事务则挂起，事务肯定不生效啊！这种属于自己配错的情况，如果真要举例，面试官也不爱听的！在面试中，一句"配置错误也会导致事务不生效，例如xxx配置，举一两个即可！"

*4、Spring的事务隔离和数据库的事务隔离是一个概念么？* OK，是一回事！ 我们先明确一点，数据库一般有四种隔离级别 数据库有四种隔离级别分别为

- read uncommitted（未提交读）
- read committed（提交读、不可重复读）
- repeatable read（可重复读）
- serializable（可串行化）

而spring只是在此基础上抽象出一种隔离级别为default，表示以数据库默认配置的为主。例如，mysql默认的事务隔离级别为repeatable-read。而Oracle 默认隔离级别为读已提交。

于是乎，有一个经典问题是这么问的

> **我数据库的配置隔离级别是Read Commited,而Spring配置的隔离级别是Repeatable Read，请问这时隔离级别是以哪一个为准？**

OK，以Spring配置的为准。JDBC有一个接口是这样的：

![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZff6o2LPrF9bqwD5aZia3elS8m3kCYKiaqjA8ianpn4KB4X0S7dHCkU8TyGPuBrN1iaW7Bp2Cq3rJ7OkJQ/640)

```java
void setTransactionIsolation(int level) throws SQLException;
```

该接口用来设置事务的隔离级别。 那么在`DataSourceUtils`中，有一段代码是这样的

他的意思就是,如果spring定义的隔离级别和数据库的不一样，则以spring定义的为准。 另外，如果spring设置的隔离级别数据库不支持，效果取决于数据库。

*5、spring事务控制放在service层，在service方法中一个方法调用service中的另一个方法，默认开启几个事务？* 此题考查的是spring的事务传播行为 我们都知道，默认的传播行为是PROPAGATION_REQUIRED，如果外层有事务，则当前事务加入到外层事务，一块提交，一块回滚。如果外层没有事务，新建一个事务执行！ 也就是说，默认情况下只有一个事务！

当然这种时候如果面试官继续追问其他传播行为的情形，如何回答？

那我们应该？我们应该？把每种传播机制都拿出来讲一遍？没必要，这种时候直接掀桌子走人。因为你就算背下来了，过几天还是忘记。用到的时候，再去查询即可。

*6、怎么保证spring事务内的连接唯一性？* 这道题很多种问法，例如Spring 是如何保证事务获取同一个Connection的?

OK,开始我们的讲解！其实答案只有一句话，因为那个Connection在事务开始时封装在了ThreadLocal里，后面事务执行过程中，都是从ThreadLocal中取的，肯定能保证唯一，因为都是在一个线程中执行的!

至于代码。。。以JDBCTemplate的execute方法为例，看看下面那张图就懂了。

![](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZff6o2LPrF9bqwD5aZia3elS8Xdpw2zOWQ3cnhR4YfKLjjb8KFdNibbPxrNlvoe40hlovib10KvAg4gow/640)

[查看原文](https://mp.weixin.qq.com/s/xfixkZRCuPKlcEwsAdlU6Q) 或 [这里](https://cloud.tencent.com/developer/article/1656547)

