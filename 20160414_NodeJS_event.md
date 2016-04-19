NodeJS众多原生模块中，其中一个重要的模块便是events模块，而且很多其他原生模块的实现也是依赖了该模块。从[上一篇](http://blog.myitlove.com/webxi-lie-002-nodejsmo-kuai-ji-zhi/)文章我们知道，一个模块中只有定义在exports对象上的方法才能被外部调用。那events模块exports出来了什么样的接口，在这篇文章中深入探究。

## NodeJS的事件机制说明

NodeJS能够在众多的后端JavaScript技术之中脱颖而出，正是因其基于事件的特点而受到欢迎。NodeJS的事件机制的特色在于：基于V8引擎实现的事件驱动IO，利用异步IO突破单线程编程模型的性能瓶颈，而且统一了前端后端的编程模型。（前端编程中，异步事件应用十分广泛，如DOM上的各种事件、Ajax事件等等）

## EventEmitter

官方API events文档[在此](https://nodejs.org/api/events.html)

从文档中可以知道events模块exports的是EventEmitter，所以我们可以：

    const EventEmitter = require('events');
    
EventEmitter是一个简单的事件监听器的实现，具有以下接口

1. emitter.addListener(eventName, listener)：
   同emitter.on(eventName, listener)
2. emitter.getMaxListeners()
3. emitter.listenerCount(eventName)
4. emitter.listeners(eventName)
5. emitter.on(eventName, listener)
6. emitter.once(eventName, listener)
7. emitter.removeAllListeners([eventName])
8. emitter.removeListener(eventName, listener)
9. emitter.setMaxListeners(n)

**使用方式一**：注册事件、触发事件

    var emitter = new EventEmitter();
    // 注册事件
    emitter.on("hello", function(someone){
        console.log("hello ",someone);
    });
    // 触发事件
    emitter.emit("hello","world");
    
**使用方式二**：继承EventEmitter

大多数时候我们不会直接使用 EventEmitter，而是在对象中继承它。包括 fs、net、 http 在内的，只要是支持事件响应的核心模块都是 EventEmitter 的子类。

为什么要这样做呢？原因有两点：

1. 首先，具有某个实体功能的对象实现事件符合语义， 事件的监听和发射应该是一个对象的方法。
2. 其次 JavaScript 的对象机制是基于原型的，支持 部分多重继承，继承 EventEmitter 不会打乱对象原有的继承关系

        var util = require('util');
        
        function Stream() {
            events.EventEmitter.call(this);
        }
        util.inherits(Stream, events.EventEmitter);
        
        var stream = new MyStream();
        stream.on("data", function(data){
            console.log('Received data:"' + data + '"');  
        });
        stream.emit("data","It works~");
        
    
**使用方式三**：多事件协作

在后端编程中，很多场景要获取多个数据源，最终合并数据渲染至客户端，如：

    api.getUser("username", function (profile) {
        // Got the profile
    });
    api.getTimeline("username", function (timeline) {
        // Got the timeline
    });
    api.getSkin("username", function (skin) {
        // Got the skin
    });
    
上述三个请求将异步进行，但为了达到三个请求都得到结果后才进行下一个步骤，程序也许会变成以下：

    api.getUser("username", function (profile) {
        api.getTimeline("username", function (timeline) {
            api.getSkin("username", function (skin) {
                // TODO
            });
        });
    });
    
这样导致请求变为串行进行，无法最大化利用，此处可以用[EventProxy](https://github.com/JacksonTian/eventproxy) 来实现多事件协作：

    var proxy = new EventProxy();
    proxy.all("profile", "timeline", "skin", function (profile, timeline, skin) {
        // TODO
    });
    api.getUser("username", function (profile) {
        proxy.emit("profile", profile);
    });
    api.getTimeline("username", function (timeline) {
        proxy.emit("timeline", timeline);
    });
    api.getSkin("username", function (skin) {
        proxy.emit("skin", skin);
    });
  
如果存在侦听器过多，引发警告，需要调用setMaxListeners(0)移除掉警告，或者设更大的警告阀值

**使用方式四**：利用事件队列解决雪崩问题

所谓雪崩问题，是在缓存失效的情景下，大并发高访问量同时涌入数据库中查询，数据库无法同时承受如此大的查询请求，进而往前影响到网站整体响应缓慢。那么在Node.js中如何应付这种情景呢。

正常访问：

    var select = function (callback) {
        db.select("SQL", function (results) {
            callback(results);
        });
    };
    
如果站点刚好启动，这时候缓存中是不存在数据的，而如果访问量巨大，同一句SQL会被发送到数据库中反复查询，影响到服务的整体性能。一个改进是添加一个状态锁并引入事件队列：

    var proxy = new EventProxy();
    var status = "ready";
    var select = function (callback) {
        proxy.once("selected", callback);
        if (status === "ready") {
            status = "pending";
            db.select("SQL", function (results) {
                proxy.emit("selected", results);
                status = "ready";
            });
        }
    };
        
由于Node.js单线程执行的原因，此处无需担心状态问题。这种方式其实也可以应用到其他远程调用的场景中，即使外部没有缓存策略，也能有效节省重复开销。

## error事件

EventEmitter 定义了一个特殊的事件 error，它包含了错误的语义，我们在遇到 异常的时候通常会触发 error 事件。
当 error 被触发时，EventEmitter 规定如果没有响 应的监听器，Node.js 会把它当作异常，退出程序并输出错误信息。
我们一般要为会触发 error 事件的对象设置监听器，避免遇到错误后整个程序崩溃

---
**参考**

1. http://www.infoq.com/cn/articles/tyq-nodejs-event
2. http://www.runoob.com/nodejs/nodejs-event.html