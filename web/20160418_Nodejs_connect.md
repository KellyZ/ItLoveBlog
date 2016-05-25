NodeJS原生的http模块提供了构建基本的http服务的功能，但更复杂的http web服务功能不够，其中第三方connect模块是众多web框架中的佼佼者，包括著名的express框架实际上也是依赖connect创建而成。

connect框架是Node的一个中间件框架，自身十分简单，其作用是基于web服务器做中间件管理，它的处理模型仅仅只是一个中间队列，进行流式处理而已。

官方地址：https://github.com/senchalabs/connect

## 概念

NodeJS HTTP模块基于事件处理网络访问无外乎两个主要的因素，请求和响应。而请求和响应有很多业务规则的处理，而且大部分的处理是可以重用的。Connect的中间件就是是扮演这样一个角色，处理请求，然后响应客户端或是让下一个中间件继续处理，每一个中间件都是重复利用并可以不断维护。

比如常见的处理有：body数据格式解析，传输压缩/解压缩，cookie设置，缓存策略，session管理，防跨站攻击crsf等等。

## 使用方法

Connect模块export的方法如下：

    app.use = function(route, fn)
    
route默认为"/"，中间件使用类似如下：

    var app = connect();
    // Middleware
    app.use(connect.staticCache());
    app.use(connect.static(__dirname + '/public'));
    app.use(connect.cookieParser());
    app.use(connect.session());
    app.use(connect.query());
    app.use(connect.bodyParser());
    app.use(connect.csrf());
    app.use(function (req, res, next) {
      // 中间件
    });
    app.listen(3001);
    
值得注意的是，必须要有一个中间件调用res.end()方法来告知客户端请求已被处理完成，否则客户端将一直处于等待状态。

## 官方维护中间件

1. body-parser - previous bodyParser, json, and urlencoded. You may also be interested in:
    * body
    * co-body
    * raw-body
2. compression - previously compress
3. connect-timeout - previously timeout
4. cookie-parser - previously cookieParser
5. cookie-session - previously cookieSession
6. csurf - previously csrf
7. errorhandler - previously error-handler
8. express-session - previously session
9. method-override - previously method-override
10. morgan - previously logger
11. response-time - previously response-time
12. serve-favicon - previously favicon
13. serve-index - previously directory
14. serve-static - previously static
15. vhost - previously vhost

这是一张Connect中间件网络参考图：

![中间件网络参考图](./images/A7V3Ej.png!web)

1. Pre-Request 通常用来改写request的原始数据
2. Request/Response 大部分中间件都在这里，功能各异
3. Post-Response 全局异常处理，改写response数据等

## logger中间件：

官方文档： http://www.senchalabs.org/connect/logger.html
   
示例：
   
    connect.logger() // default
    connect.logger('short')
    connect.logger('tiny')
    connect.logger({ immediate: true, format: 'dev' })
    connect.logger(':method :url - :referrer')
    connect.logger(':req[content-type] -> :res[content-type]')
    connect.logger(function(tokens, req, res){ return 'some format string' })

## body-parser

描述：请求内容解析中间件

官方文档：

1. https://www.npmjs.com/package/body-parser
2. http://www.senchalabs.org/connect/bodyParser.html
   
目前body-parser支持：

1. JSON body parser： application/json 
2. Raw body parser
3. Text body parser
4. URL-encoded form body parser: application/x-www-form-urlencode

另外multipart会在connect 3中被移除；

那bodyParser有哪些接口？

    var bodyParser = require('body-parser')
   
1. bodyParser.json(options)
2. bodyParser.raw(options)
3. bodyParser.text(options)
4. bodyParser.urlencoded(options)



## 参考

1. https://github.com/senchalabs/connect/wiki
2. http://www.infoq.com/cn/news/2011/11/tyq-nodejs-static-file-server
3. http://www.tuicool.com/articles/emeuie