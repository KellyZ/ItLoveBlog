NodeJS第三方connect模块以插件的形式构建了一个灵活的web
框架，另一个第三方模块express在connect的基础上构建了一个更为强大灵活的web应用程序框架。

[官方](http://expressjs.com/)描述：Fast, unopinionated, minimalist web framework for Node.js （高度包容、快速而极简的web框架）

express包含以下功能：

1. 路由： routing,4.0开始成了一个单独的组件Express.Router
2. 模版引擎： template
3. 错误处理：Error handling
4. 中间件：使用connect

express通过use方法注册中间件，同connect。

## 模板引擎

主要流行的模板引擎有：

1. hbs: Handlebars
2. jade
3. ejs

## 路由Express.Router

Express.Router是一个构造函数，调用后返回一个路由器实例。然后，使用该实例的HTTP动词方法，为不同的访问路径，指定回调函数；最后，挂载到某个路径。

    var router = express.Router();

    router.get('/', function(req, res) {
      res.send('首页');
    });
    
    router.get('/about', function(req, res) {
      res.send('关于');
    });
    
    app.use('/', router);

**route方法**可以接受访问路径作为参数

    var router = express.Router();

    router.route('/api')
    	.post(function(req, res) {
    		// ...
    	})
    	.get(function(req, res) {
    		Bear.find(function(err, bears) {
    			if (err) res.send(err);
    			res.json(bears);
    		});
    	});
    
    app.use('/', router);
    
**use方法**为router对象指定中间件，即在数据正式发给用户之前，对数据进行处理。下面就是一个中间件的例子

    router.use(function(req, res, next) {
    	console.log(req.method, req.url);
    	next();	
    });

**param方法**用于路径参数的处理。注意，param方法必须放在HTTP动词方法之前。

    router.param('name', function(req, res, next, name) {
    	// 对name进行验证或其他处理……
    	console.log(name);
    	req.name = name;
    	next();	
    });
    
    router.get('/hello/:name', function(req, res) {
    	res.send('hello ' + req.name + '!');
    });
    
## Error Hander



## 参考

1. http://javascript.ruanyifeng.com/nodejs/express.html
2. 