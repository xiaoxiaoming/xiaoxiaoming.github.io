---
layout: post
title: "领域驱动设计之模型"
date: 2013-11-10 13:50
comments: true
categories: 
---

相比各种model，领域模型是充血，有各种职责。一般的解决方式有：
 
 * 利用Spring容器，将Component注入到模型
 * 利用语言的mixin特性，在运行期混Component。
  

这两种方式都能解决问题，但是存在几个问题。

  * 需要依赖部件，才能对Model进行测试，如果逻辑依赖Dao，那必须为mock或者stub。
  * 越是复杂的逻辑，越难改动。
  * 

从从另外一个角度来看，要接上述问题，需要对模型的边界进行划分，如图所示


<pre>

                --------------------------
                |           DB            |
                |    -----------------    |
                |    |    EventBus   |    |
                |    |    -------    |    |
                |Mail|    |Domain|   | MQ |
                |    |    --------   |    |
                |    |    EventBus   |    |
                |    -----------------    |
                |         Cache           |
                --------------------------
</pre>

最核心的中间的领域模型，接受外界输入的各种命令，以及相关数据，经过一系列逻辑处理后，讲变化的结果，以事件的形式，通过EventBus，通知给外围。

具体的实践如下：

1、定义Domian的抽象类AggregateRoot，解决Domain与外围的通讯，Domain与内部的通讯。

为此，定义了两个函数：
public void raised(DomainEvent<T> e);
public void handleThis(DomainEvent<T> e);

raised函数作用就是讲事件通知给依赖的部件。
handleThis是个特殊函数，它会将领域事件通知给Domain模型，为什么会有此设置，目前是为了能通过过事件追朔模型的状态。

。。另外需要依赖：
 * EventBus
 * EventStore
 * EventCache
 * ClassCache


EventBus：  讲事件通过外围，执行方式为同步，目的是保证事务的一致性。
EventStore：将发布的事件存储到其他介质中，比如db，file，logback。
EventCache：讲Domain最新状态写到缓存中，以便下次获取，提高性能
ClassCache：Domian发出的Event后，会以反射的形式找到对应的handle函数，利用ClassCache保存class对象，提高性能

根据实际情况，

2、定义一个Domian类，实现其逻辑，如
```java
public class Account extends AggregateRoot{
    private Long id;
    private Long loginCount =0
    public Account(Long id){
        this.raised(new AccountCreatedEvent(uid));
    }
    public void handle(AccountCreatedEvent e){
        //this.id = e.getid();
    }
    public void login(){
        this.raised(new AccountLoginEvent(uid));
    }
    public void handle(AccountLoginEvent e){
        loginCount++;
    }
}
```
3、调用领域模型，
```java
  //创建一个用户
  new Account(1L)
```
利用此方式，就可创建一个Account对象，并保存到数据库中。

```java
  //创建一个用户,并登陆
  new Account(1L).login();
```
利用此方式，就可创建一个Account对象，并统计其登陆次数。


就先说到这里。
在接下来的几篇，对针EventBus，EventStore，EventCache等特性进行讲解。


