---
title: 事件监听/发布 MVP 版实现
layout: default
nav_order: 3
permalink: /mvp_observer_design_pattern
has_children: false
parent: Java
---
# 观察者模式-事件监听/发布 MVP 版实现

> 实现简易事件发送/事件监听

## 概览

1. 自定义监听注解类 `CustomEventLister` 作用于方法上，用于标识指定方法可监听事件消息

2. 定义观察者，并在需要监听的方法上声明监听注解 `@CustomEventLister`

3. 自定义  `CustomEventAction` 

   - `target` 监听对象
   - `method` 监听方法

   - `execute()` 方法，通过反射调用指定方法

4. 观察者管理：维护一个线程安全的 Map 存储  “消息类型”—“action集合” 列表

   - 注册方法，观察者注册时调用，扫描观察者类，识别存在 `@CustomEventLister`注解的方法并进行注册
   - 获取指定消息类型的 `CustomEventAction` 集合方法，当发送消息时获取消息类型 `CustomEventAction` 集合并执行

5. 自定义事件执行管理（支持同步阻塞/异步非阻塞执行）

   - 注册观察者
   - 发送事件消息
     - 同步阻塞执行
     - 异步非阻塞执行

## 自定义注解

~~~java
@Retention(RetentionPolicy.RUNTIME)
@Target(value = ElementType.METHOD)
public @interface CustomEventLister {
}
~~~

## 自定义 Action 实体类

- 重写 `equals` `hashCode` 方法，避免在观察者管理中重复注册导致最终 `CustomEventAction `出现重复

~~~java
public class CustomEventAction {
    private final Method method;
    private final Object target;

    public CustomEventAction(Object target, Method method) {
        if (target == null || method == null) {
            throw new RuntimeException("CustomEventAction() target or method param is null.");
        }
        this.target = target;
        this.method = method;
        this.method.setAccessible(true);
    }
  
    public void execute(Object event) {
        try {
            this.method.invoke(target, event);
        } catch (IllegalAccessException | InvocationTargetException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof CustomEventAction)) return false;
        CustomEventAction that = (CustomEventAction) o;
        return Objects.equals(method, that.method) && Objects.equals(target, that.target);
    }

    @Override
    public int hashCode() {
        return Objects.hash(method, target);
    }
}
~~~

## 核心类-观察者管理（注册/查询）

- 使用线程安全类进行存储已注册的观察者与 `CustomEventAction` 集合

~~~java
@Component
public class ObserverRegistry {
    // key-消息类型，value-执行方法集合
    private final ConcurrentHashMap<Class<?>, CopyOnWriteArraySet<CustomEventAction>> registry = new ConcurrentHashMap<>();

    public void register(Object observer) {
        // 查询所有观察者监听方法
        Map<Class<?>, List<CustomEventAction>> customEventActions = findAllObserverActions(observer);
        // 遍历监听方法并进行注册
        for (Map.Entry<Class<?>, List<CustomEventAction>> map : customEventActions.entrySet()) {
            Class<?> type = map.getKey();
            List<CustomEventAction> actions = map.getValue();
            CopyOnWriteArraySet<CustomEventAction> sets = registry.get(type);
            if (sets == null) {
                registry.put(type, new CopyOnWriteArraySet<>());
                sets = registry.get(type);
            }
            sets.addAll(actions);
        }
    }

    public List<CustomEventAction> getMatchedActions(Object event) {
        List<CustomEventAction> matchedObservers = new ArrayList<>();
        Class<?> postedEventType = event.getClass();
        for (Map.Entry<Class<?>, CopyOnWriteArraySet<CustomEventAction>> entry : registry.entrySet()) {
            Class<?> type = entry.getKey();
            CopyOnWriteArraySet<CustomEventAction> actions = entry.getValue();
            // 兼容多态情况判断
            if (postedEventType.isAssignableFrom(type)) {
                matchedObservers.addAll(actions);
            }
        }
        return matchedObservers;
    }

    public int getRegistrySize(){
        return registry.size();
    }
  
    public Map<Class<?>, List<CustomEventAction>> findAllObserverActions(Object observer) {
        // key-参数类型，value-监听方法集合
        Map<Class<?>, List<CustomEventAction>> ret = new HashMap<>();
        Class<?> clazz = observer.getClass();
        // 获取所有带 @CustomEventLister 注解的方法
        List<Method> methods = getAnnotatedMethods(clazz);
        for (Method method : methods) {
            // 获取方法参数 只支持单个参数
            Class<?>[] parameterTypes = method.getParameterTypes();
            Class<?> type = parameterTypes[0];
            List<CustomEventAction> customEventActions = ret.get(type);
            if (customEventActions == null) {
                ret.put(type, new ArrayList<>());
                customEventActions = ret.get(type);
            }
            customEventActions.add(new CustomEventAction(observer, method));
        }
        return ret;
    }
    private List<Method> getAnnotatedMethods(Class<?> clazz) {
        List<Method> ret = new ArrayList<>();
        Method[] methods = clazz.getDeclaredMethods();
        for (Method method : methods) {
            if (method.isAnnotationPresent(CustomEventLister.class)) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length != 1) {
                    throw new RuntimeException("Method must one parameter");
                }
                ret.add(method);
            }
        }
        return ret;
    }
}
~~~

## 观察者/监听者

~~~java
@Slf4j
@Component
public class CustomEventOberver {
    @CustomEventLister
    public void handleEventMsg(String msg) {
        log.info("## CustomEventTest handleEventMsg method msg={} ##", msg);
    }

    @CustomEventLister
    public void handleEventCustomMsg(CustomMsg msg) {
        log.info("## CustomEventTest handleEventCustomMsg method msg={} ##", msg.toString());
    }
}
~~~

## 自定义事件类

### 同步阻塞

- 其中 `init()` 方法注册观察者

~~~java
@Component
public class CustomEventBus {
    private Executor executor;
    @Resource
    private ObserverRegistry observerRegistry;

    @Resource
    private CustomEventOberver customEventOberver;

    @PostConstruct
    private void init() {
        observerRegistry.register(customEventOberver);
    }

    public CustomEventBus() {
    }

    protected CustomEventBus(Executor executor) {
        this.executor = executor;
    }

    protected void setExecutor(Executor executor) {
        this.executor = executor;
    }

    public void register(Object object) {
        observerRegistry.register(object);
    }

    public void post(Object event) {
        List<CustomEventAction> actions = observerRegistry.getMatchedActions(event);
        for (CustomEventAction action : actions) {
            if (executor != null) {
                executor.execute(() -> action.execute(event));
            } else {
                action.execute(event);
            }
        }
    }
}
~~~

### 异步非阻塞

- 支持自定义线程池

~~~java
@Component
public class CustomAsyncEventBus extends CustomEventBus {
    public CustomAsyncEventBus(Executor executor) {
        super(executor);
    }

    public CustomAsyncEventBus() {
    }

    @Override
    public void setExecutor(Executor executor) {
        super.setExecutor(executor);
    }
}
~~~

## 测试并发送事件

- 依赖注入同步自定义事件执行对象、异步自定义事件执行对象、线程池
- `init()` 依赖注入完成后使用该方法设置异步自定义事件执行线程池

~~~java
@RestController
@RequestMapping("/custom_event")
public class CustomEventController {

    @Resource
    private CustomEventBus customEventBus;

    @Resource
    private ThreadPoolTaskExecutor threadPoolTaskExecutor;

    @Resource
    private CustomAsyncEventBus customAsyncEventBus;

    @PostConstruct
    private void init() {
        customAsyncEventBus.setExecutor(threadPoolTaskExecutor);
    }

    @GetMapping
    public String customEventHandler() {
        customEventBus.post("custom_event string msg");
        CustomMsg customMsg = new CustomMsg();
        customMsg.setName("testName");
        customMsg.setUserId("testUserId");
        customEventBus.post(customMsg);

        customAsyncEventBus.post("custom_event string async msg");
        customMsg.setName("testAsyncName");
        customAsyncEventBus.post(customMsg);
        return "custom_event success";
    }
}
~~~

## 启动 Spring Boot 应用并调用

- 执行

  ```shell
  curl http://localhost:9820/custom_event
  ```

- 控制台输出

  - nio-9820-exec-${i} 同步阻塞执行输出日志
  - task-${i} 异步阻塞执行输出日志，线程池 `ThreadPoolTaskExecutor`

  ~~~sh
  2023-03-20 18:28:58.003  INFO 41471 --- [nio-9820-exec-2] c.c.d.o.m.observer.CustomEventOberver    : ## CustomEventTest handleEventMsg method msg=custom_event string msg ##
  2023-03-20 18:28:58.005  INFO 41471 --- [nio-9820-exec-2] c.c.d.o.m.observer.CustomEventOberver    : ## CustomEventTest handleEventCustomMsg method msg=CustomMsg(userId=testUserId, name=testName) ##
  2023-03-20 18:28:58.005  INFO 41471 --- [         task-1] c.c.d.o.m.observer.CustomEventOberver    : ## CustomEventTest handleEventMsg method msg=custom_event string async msg ##
  2023-03-20 18:28:58.005  INFO 41471 --- [         task-2] c.c.d.o.m.observer.CustomEventOberver    : ## CustomEventTest handleEventCustomMsg method msg=CustomMsg(userId=testUserId, name=testAsyncName) ##
  ~~~

## 完整代码地址

- [Github](https://github.com/szclouds/clouds2089/tree/main/src/main/java/com/clouds/designPattern/observer/mvpEvent)
