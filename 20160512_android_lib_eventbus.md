https://github.com/greenrobot/EventBus

## 简介

官方特性声明：

1. simplifies the communication between components
   * decouples event senders and receivers
   * performs well with Activities, Fragments, and background threads
   * avoids complex and error-prone dependencies and life cycle issues
2. makes your code simpler
3. is fast
4. is tiny (~50k jar)
5. is proven in practice by apps with 100,000,000+ installs
6. has advanced features like delivery threads, subscriber priorities, etc

架构图如下：

![](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/EventBus-Publish-Subscribe.png)

## 三步使用

1. Define events:

        public class MessageEvent { /* Additional fields if needed */ }

2. Prepare subscribers： Register your subscriber (in your onCreate or in a constructor):

        eventBus.register(this);

    Declare your subscribing method:
   
        @Subscribe
        public void onEvent(AnyEventType event) {/* Do something */};
    
3. Post events:
    
        eventBus.post(event);
        
## 四种线程模型

EventBus包含4个ThreadMode：PostThread，MainThread，BackgroundThread，Async

对应的Event订阅触发方法名为：onEventPostThread， onEventMainThread，onEventBackgroundThread，onEventAsync

1. ThreadMode: MAIN 在UI线程执行
    
        // Called in Android UI's main thread
        @Subscribe(threadMode = ThreadMode.MAIN)
        public void onMessage(MessageEvent event) {
            textField.setText(event.message);
        }

2. ThreadMode: POSTING 在当前发布事件的线程执行

        @Subscribe(threadMode = ThreadMode.POSTING) // ThreadMode is optional here
        public void onMessage(MessageEvent event) {
            log(event.message);
        }

3. ThreadMode: BACKGROUND，如果事件是在UI线程中发布出来的，那么就会在子线程中运行，如果事件本来就是子线程中发布出来的，那么直接在该子线程中执行。

        // Called in the background thread
        @Subscribe(threadMode = ThreadMode.BACKGROUND)
        public void onMessage(MessageEvent event){
            saveToDisk(event.message);
        }

4. ThreadMode: ASYNC，无论事件在哪个线程发布，都会创建新的子线程在执行

        // Called in a separate thread
        @Subscribe(threadMode = ThreadMode.ASYNC)
        public void onMessage(MessageEvent event){
            backend.send(event.message);
        }

## 订阅

    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }

扫描订阅者class中以onEvent开头的方法：

    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
    
## 触发

    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
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

## 事件取消

事件取消往往是由优先级高的事件订阅者取消

    // Called in the same thread (default)
    @Subscribe
    public void onEvent(MessageEvent event){
        // Process the event
        …
        
        EventBus.getDefault().cancelEventDelivery(event) ;
    }

## Sticky Events

    EventBus.getDefault().postSticky(new MessageEvent("Hello everyone!"));

sticky event是在订阅者处理之前的一段时间发出的，比如一个新的Activity启动后订阅可以收到之前发出的sticky event：

    @Override
    public void onStart() {
        super.onStart();
        EventBus.getDefault().register(this);
    }

    @Subscribe(sticky = true, threadMode = ThreadMode.MAIN)
    public void onEvent(MessageEvent event) {
        // UI updates must run on MainThread
        textField.setText(event.message);
    }

    @Override
    public void onStop() {
        EventBus.getDefault().unregister(this);
        super.onStop();
    }
    
某些情况下又需要撤销sticky event:

    MessageEvent stickyEvent = EventBus.getDefault().removeStickyEvent(MessageEvent.class);
    // Better check that an event was actually posted before
    if(stickyEvent != null) {
        // Now do something with it
    }

## 参考

1. http://blog.csdn.net/lmj623565791/article/details/40794879
2. http://blog.csdn.net/lmj623565791/article/details/40920453
3. http://greenrobot.org/eventbus/documentation/delivery-threads-threadmode/