#### EventBus

EventBus是一个Android事件发布/订阅框架，通过解耦发布者和订阅者简化 Android 事件传递。EventBus使用简单,并将事件发布和订阅充分解耦,从而使代码更简洁



##### 使用

###### 注册订阅者,取消注册

```
EventBus.getDefault().register(Object subscriber);
EventBus.getDefault().unregister(Object subscriber)
```

###### 响应事件订阅方法

```
//3.0版本
@Subscribe(threadMode = ThreadMode.BACKGROUND, sticky = true, priority = 100)
public void test(TestEvent str) {
    
}
```

###### 发送事件

```
EventBus.getDefault().post(new TestEvent("string"))
```



##### 注册分析

首先Eventbus采用双重校验并加锁的单例模式生成EventBus实例，getDefault返回唯一的实例。

```
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

注册首先获取获取了订阅者的class对象，然后通过`findSubscriberMethods`返回订阅方法SubscriberMethod的集合，最后统一都调用subscribe订阅方法

```
SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;
    ...
}
```

###### findSubscriberMethods的实现

`SubscriberMethodFinder`类就是用来查找和缓存订阅者响应函数的信息的类。所以我们首先要知道怎么能获得订阅者响应函数的相关信息

```
 private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
 
 List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //先从METHOD_CACHE取看是否有缓存,key:保存订阅类的类名,value:保存类中订阅的方法数据,
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        //是否忽略注解器生成的MyEventBusIndex类
        if (ignoreGeneratedIndex) {
            //利用反射来读取订阅类中的订阅方法信息
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            //从注解器生成的MyEventBusIndex类中获得订阅类的订阅方法信息
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            //保存进METHOD_CACHE缓存
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

这个方法，首先去看下缓存METHOD_CACHE中是否有对应的订阅方法，如果有，直接返回。没有的话，通过注解处理器或者反射来获取。同时放入METHOD_CACHE中。

###### 反射

虽然ignoreGeneratedIndex默认返回false。但是还是学习下反射的方式

```
    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        //FindState 用来做订阅方法的校验和保存
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            //通过反射来获得订阅方法信息
            findUsingReflectionInSingleClass(findState);
            //查找父类的订阅方法
            findState.moveToSuperclass();
        }
        //获取findState中的SubscriberMethod(也就是订阅方法List)并返回
        return getMethodsAndRelease(findState);
    }
```

这里通过`FindState`类来做订阅方法的校验和保存,并通过`FIND_STATE_POOL`静态数组来保存`FindState`对象,可以使`FindState`复用,避免重复创建过多的对象.最终是通过`findUsingReflectionInSingleClass()`来具体获得相关订阅方法的信息的

```
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        //通过反射得到方法数组
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        //遍历Method
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                //保证必须只有一个事件参数
                if (parameterTypes.length == 1) {
                    //得到注解
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        //校验是否添加该方法
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //实例化SubscriberMethod对象并添加
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

通过反射的方式获取订阅者类中的所有声明方法，然后在这些方法里面寻找以@Subscribe作为注解的方法进行处理（！！！部分的代码），先经过一轮检查，看看findState.subscriberMethods是否存在，如果没有的话，将方法名，threadMode，优先级，是否为sticky方法封装为SubscriberMethod对象，添加到subscriberMethods列表中。 

###### subscribe()方法的实现

```
 //必须在同步代码块里调用
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        //获取订阅的事件类型
        Class<?> eventType = subscriberMethod.eventType;
        //创建Subscription对象
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        //从subscriptionsByEventType里检查是否已经添加过该Subscription,如果添加过就抛出异常
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }
        //根据优先级priority来添加Subscription对象
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        //将订阅者对象以及订阅的事件保存到typesBySubscriber里.
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
        //如果接收sticky事件,立即分发sticky事件
        if (subscriberMethod.sticky) {
            //eventInheritance 表示是否分发订阅了响应事件类父类事件的方法
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

这里

```
final class Subscription {
    final Object subscriber;
    final SubscriberMethod subscriberMethod;
}
    
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;    key为eventType 也就是示例中的TestEvent
private final Map<Object, List<Class<?>>> typesBySubscriber;
   key为 订阅者对象  value为订阅的事件保存到typesBySubscriber
```



##### 取消注册

分别从`typesBySubscriber`和`subscriptions`里分别移除订阅者以及相关信息即可

```
    public synchronized void unregister(Object subscriber) {
        //通过typesBySubscriber来取出这个subscriber订阅者订阅的事件类型,
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            //分别解除每个订阅了的事件类型
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            //从typesBySubscriber移除subscriber
            typesBySubscriber.remove(subscriber);
        } else {
            Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }
```

##### POST

```
public void post(Object event) {
    // 5.1
    PostingThreadState postingState = currentPostingThreadState.get();
    List <Object> eventQueue = postingState.eventQueue;
    eventQueue.add(event);

    // 5.2
    if (!postingState.isPosting) {
        postingState.isMainThread = isMainThread();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}
```

1. currentPostingThreadState是一个ThreadLocal类型的，里面存储了PostingThreadState，而PostingThreadState中包含了一个eventQueue和其他一些标志位；

2. 然后把传入的event，保存到了当前线程中的一个变量PostingThreadState的eventQueue中

   ```
   private final ThreadLocal <PostingThreadState> currentPostingThreadState = new ThreadLocal <PostingThreadState> () {
       @Override
       protected PostingThreadState initialValue() {
           return new PostingThreadState();
       }
   };
   
   /** For ThreadLocal, much faster to set (and get multiple values). */
   final static class PostingThreadState {
       final List <Object> eventQueue = new ArrayList<>();
       boolean isPosting;
       boolean isMainThread;
       Subscription subscription;
       Object event;
       boolean canceled;
   }
   ```

3. 最后调用到postSingleEvent()方法

   ```
       private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
           Class<?> eventClass = event.getClass();
           boolean subscriptionFound = false;
           //是否触发订阅了该事件(eventClass)的父类,以及接口的类的响应方法.
           if (eventInheritance) {
               //查找eventClass类所有的父类以及接口
               List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
               int countTypes = eventTypes.size();
               //循环postSingleEventForEventType
               for (int h = 0; h < countTypes; h++) {
                   Class<?> clazz = eventTypes.get(h);
                   //只要右边有一个为true,subscriptionFound就为true
                   subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
               }
           } else {
               //post单个
               subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
           }
           //如果没发现
           if (!subscriptionFound) {
               if (logNoSubscriberMessages) {
                   Log.d(TAG, "No subscribers registered for event " + eventClass);
               }
               if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                       eventClass != SubscriberExceptionEvent.class) {
                   //发送一个NoSubscriberEvent事件,如果我们需要处理这种状态,接收这个事件就可以了
                   post(new NoSubscriberEvent(this, event));
               }
           }
       }
   ```

   我们可以很清楚的发现是在`postSingleEventForEventType()`方法里去进行事件的分发

4. postSingleEventForEventType

   ```
   private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
           CopyOnWriteArrayList<Subscription> subscriptions;
           //获取订阅了这个事件的Subscription列表.
           synchronized (this) {
               subscriptions = subscriptionsByEventType.get(eventClass);
           }
           if (subscriptions != null && !subscriptions.isEmpty()) {
               for (Subscription subscription : subscriptions) {
                   postingState.event = event;
                   postingState.subscription = subscription;
                   //是否被中断
                   boolean aborted = false;
                   try {
                       //分发给订阅者
                       postToSubscription(subscription, event, postingState.isMainThread);
                       aborted = postingState.canceled;
                   } finally {
                       postingState.event = null;
                       postingState.subscription = null;
                       postingState.canceled = false;
                   }
                   if (aborted) {
                       break;
                   }
               }
               return true;
           }
           return false;
       }
   
       private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
           switch (subscription.subscriberMethod.threadMode) {
               case POSTING:
                   invokeSubscriber(subscription, event);
                   break;
               case MAIN:
                   if (isMainThread) {
                       invokeSubscriber(subscription, event);
                   } else {
                       mainThreadPoster.enqueue(subscription, event);
                   }
                   break;
               case BACKGROUND:
                   if (isMainThread) {
                       backgroundPoster.enqueue(subscription, event);
                   } else {
                       invokeSubscriber(subscription, event);
                   }
                   break;
               case ASYNC:
                   asyncPoster.enqueue(subscription, event);
                   break;
               default:
                   throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
           }
       }
   ```

   在`postToSubscription()`通过不同的`threadMode`在不同的线程里`invoke()`订阅者的方法