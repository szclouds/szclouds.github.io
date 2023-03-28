---
title: 观察者模式
layout: default
nav_order: 2
permalink: /observer_design_pattern
has_children: false
parent: Java
---
# 观察者模式及 Spring Boot 应用中的事件发布/监听机制

> 创建型设计模式：解决对象创建的问题
>
> 结构型设计模式：解决类或对象的组合或组装问题
>
> 行为型设计模式：类或对象之间的交互问题

## 描述

观察者模式是在对象之间定义一个一对多的依赖，当一个对象的状态改变的时候，所有依赖的对象都会自动收到通知。

被依赖的对象为被观察者，依赖对象为观察者。

实现方式：

- 同步阻塞，典型的实现方式
- 异步非阻塞，实现代码解耦外，同时提高代码的执行效率
- 跨进程实现，通过消息队列实现，解耦更彻底

## 模版代码

~~~java
// 观察者接口
public interface Observer {
    void sendMsg(String message);
}

// 被观察者接口
public interface Subject {
    // 注册观察者
    void registerObserver(Observer observer);
    // 移除观察者
    void removeObserver(Observer observer);
  
    void notifyObservers(String message);
}

// 观察者1
@Slf4j
public class ObserverOne implements Observer {
    @Override
    public void sendMsg(String message) {
        log.info("ObserverOne msg={}", message);
    }
}

// 观察者2
@Slf4j
public class ObserverTwo implements Observer {
    @Override
    public void sendMsg(String message) {
        log.info("ObserverTwo msg={}", message);
    }
}

// 被观察者
@Slf4j
public class SimpleSubject implements Subject {
    private final Set<Observer> observers = new HashSet<>();

    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers(String message) {
        if (observers.size() < 0) {
            log.warn("observers size < 0");
        }
        for (Observer observer : observers) {
            observer.sendMsg(message);
        }
    }
}

// 测试
public class SimpleObserverTest {
    public static void main(String[] args) {
        SimpleSubject simpleSubject = new SimpleSubject();
        simpleSubject.registerObserver(new ObserverOne());
        simpleSubject.registerObserver(new ObserverTwo());
        simpleSubject.notifyObservers("SimpleObserverTest msg");
    }
}

~~~

完整代码参见：[观察者模式](https://github.com/szclouds/clouds2089/tree/main/src/main/java/com/clouds/designPattern/observer/simple)

## 具体业务场景：用户注册后发送优惠券及消息

1. 定义观察者接口
2. 发送优惠券+发送消息实现观察者接口，编写不同业务逻辑
3. 用户注册API接口，通过依赖注入多个观察者
4. 用户注册成功后遍历观察者，执行不同观察者业务逻辑

如上用户注册流程，实现了注册接口与其他业务逻辑的解耦，将不同行为的代码进行解耦，将观察者与被观察者解耦。借助设计模式，可以优化代码结构，将一堆代码拆分为职责单一的小类，让其满足开闭原则、高内聚松耦合等特性，以此来控制和应对代码的复杂性，提高代码的扩展性。

当注册接口对请求处理时间较为敏感时，可将观察者执行逻辑通过异步处理（线程池）。

扩展：生产者-消费者模型和观察者模式的区别：

- 生产者-消费者模型是多对多关系，异步实现
- 观察者模式是一对多关系，同步或异步实现
- 两者的作用都是用于解耦

## 异步非阻塞观察者模式实现

~~~java
// 异步非阻塞观察者模式实现，只改造 Subject 实现部分，其余部分同阻塞简单模式
// 线程池可通过依赖注入完成，具体参数可根据业务及运行环境配置
@Slf4j
public class AsyncSubject implements Subject {
    private final Set<Observer> observers = ConcurrentHashMap.newKeySet();
    private final Executor executor = new ThreadPoolExecutor(
            3,
            5,
            5,
            TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(5));

    @Override
    public void registerObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers(String message) {
        if (observers.size() < 1) {
            log.warn("observers size < 1");
        }
        for (Observer observer : observers) {
            executor.execute(() -> observer.sendMsg(message));
        }
    }
}
~~~

## Spring Boot Event Listener

1. 开启异步执行，可在启动类上增加注解：`@EnableAsync`

2. 定义事件发布者

   ~~~java
   @Component
   public class SimpleEventPublisher {
   
       @Resource
       private ApplicationEventPublisher applicationEventPublisher;
   
       public void publishMsgEvent(String msg) {
           applicationEventPublisher.publishEvent(msg);
       }
   
       public void publishCustomMsg(CustomMsg msg) {
           applicationEventPublisher.publishEvent(msg);
       }
   }
   ~~~

3. 定义事件监听者，同步阻塞和异步非阻塞方法，通过 `@EventListener`  标识

   ~~~java
   @Component
   @Slf4j
   public class SimpleEventListener {
   
       @EventListener
       public void sendEventMsg(String msg) {
           log.info("## sendEventMsg={} ##", msg);
       }
   
       @EventListener
       @Async
       public void sendCustomMsg(CustomMsg msg) {
           log.info("## sendCustomMsg={} ##", msg.toString());
       }
   }
   ~~~

4. 发布事件消息

   ~~~java
   @RestController
   @RequestMapping("/simple_event")
   public class SimpleEventController {
       @Resource
       private SimpleEventPublisher simplePublisher;
   
       @GetMapping
       public String simpleEventHandler() {
           simplePublisher.publishMsgEvent("string msg");
           CustomMsgChild customMsg = new CustomMsgChild("name", 100);
           simplePublisher.publishCustomMsg(customMsg);
           return "simpleEventHandler test success";
       }
   }
   ~~~

5. [代码链接](https://github.com/szclouds/clouds2089/tree/main/src/main/java/com/clouds/designPattern/observer/springEvent)

注意事项：

- 事件发布者可发布多个事件类型
- 事件监听者可监听多个类型，最终执行将会通过类型（多态的情况下同适用）进行匹配
- 事件监听者可声明同步或异步执行
- 异步执行线程池支持自定义配置
