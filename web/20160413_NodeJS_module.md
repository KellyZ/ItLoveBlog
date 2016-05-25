从[上一篇文](http://blog.myitlove.com/commonjs/)章中，我们知道JavaScript进化的过程中，非常重要的一个里程碑是CommonJS定制相关的规范，其中一个重要的就是**模块**，这篇文章就来研究下NodeJS对模块的实现。

## Node.js的require机制

NodeJS的模块分为两类，一类为原生模块，可参考https://nodejs.org/api/ ，一类为文件模块。原生模块在NodeJS源代码编译的时候编译进了二进制执行文件，加载的速度最快。另一类文件模块是动态加载的。但是NodeJS对原生模块和文件模块都进行了缓存，于是在第二次require时是不会有重复开销的。其中原生模块都被定义在lib目录下。

文件模块加载的工作主要由原生模块**module**来实现和完成，该原生模块在启动时已经被加载。

比如我们运行**node app.js**，其实就让module模块加载app.js文件模块，执行module.runMain静态方法，如下：

    // bootstrap main module.
    Module.runMain = function () {
    // Load the main module--the command line argument.
    Module._load(process.argv[1], null, true);
    };

   其中_load静态方法在分析文件名之后执行：
    
     var module = new Module(id, parent);
     module.load(filename);
     
   load加载模块，NodeJS会根据后缀名来决定加载方法：
   
   1. .js。通过fs模块同步读取js文件并编译执行；
   2. .node。通过C/C++进行编写的Addon。通过dlopen方法进行加载；
   3. .json。读取文件，调用JSON.parse解析加载；

   其中在编译js后缀的模块文件时实际会对js文件中内容进行包装，包装的app.js会变成如下形式：
   
    (function (exports, require, module, __filename, __dirname) {
        // app.js的原始内容
    });
    
   返回一个具体的function对象，运行该function对象传入module对象的exports、require方法、module、文件名、目录名，这也是为什么app.js中有exports、require、module这些变量。（module变量是这个模块对象自身，exports是在module的构造函数中初始化的一个空对象{}，而不是null）
   
   load方法在载入、编译、缓存了module后，返回module的exports对象，这也是为什么只有定义在exports对象上的方法才能被外部调用的原因。（模块载入机制定义在lib/module.js文件中）
   
   模块加载的优先级：缓存模块 > 原生模块 > 文件模块。

## 文件模块的路径

   对于每一个被加载的文件模块，创建这个模块对象的时候，这个模块便会有一个paths属性，即module.paths，可以通过如下方式查看：
   
    console.log(module.paths);
    
   除此之外还有一个全局module path，是当前node执行文件的相对目录（../../lib/node),如果在环境变量中设置HOME目录和NODE_PATH目录，整个路径还会包含这两个目录下的.node_libraries于.node_modules。
   
   根据相关路径进行查找，如果查找不到文件，将尝试将require的参数作为一个包来进行查找，读取目录下的package.json文件，取得main参数指定的文件

## NPM基于包规范

   CommonJS定义了JavaScript的包结构规范，而NPM的出现则是实现CommonJS的包规范，解决了包的安装卸载、依赖管理、版本管理等问题。
   
   一个符合CommonJS规范的包应该是如下这种结构：
   
   1. 一个package.json文件应该存在于包顶级目录下；
   2. 二进制文件应该包含在bin目录下
   3. JavaScript代码应该包含在lib目录下
   4. 文档应该在doc目录下
   5. 单元测试应该在test目录下
   
   从上面require的查找过程可以知道，NodeJS在没有找到目标模块文件时，会将当前目录当做一个包来尝试加载，所以在package.json文件中最重要的一个字段就是**main**。对于require只需要main属性即可，但实际除此之外包管理还需要接受安装、卸载、依赖管理、版本管理等流程，所以package.json文件还定义了如下一些必须的字段：
   
   1. name。包名，需要在NPM上是唯一的。不能带有空格
   2. description。包简介。通常会显示在一些列表中
   3. version。版本号。一个语义化的版本号，通常为x.y.z。该版本号十分重要，常常用于一些版本控制的场合
   4. keywords。关键字数组。用于NPM中的分类搜索
   5. maintainers。包维护者的数组。数组元素是一个包含name、email、web三个属性的JSON对象
   6. contributors。包贡献者的数组。第一个就是包的作者本人。
   7. bugs。一个可以提交bug的URL地址。可以是邮件地址
   8. licenses。包所使用的许可证
   9. repositories。托管源代码的地址数组
   10. dependencies。当前包需要的依赖。这个属性十分重要，NPM会通过这个属性，帮你自动加载依赖的包

   一些额外的字段，如bin、scripts、engines、devDependencies、author。这里可以重点提及一下scripts字段，scripts字段的对象指明了在进行操作时运行哪个文件，或者执行拿条命令。如下为一个较全面的scripts案例：
   
    "scripts": {
        "install": "install.js",
        "uninstall": "uninstall.js",
        "build": "build.js",
        "doc": "make-doc.js",
        "test": "test.js",
    }
    
## Node.js模块与前端模块的异同
    
   通常有一些模块可以同时适用于前后端，但是在浏览器端通过script标签的载入JavaScript文件的方式与Node.js不同。Node.js在载入到最终的执行中，进行了包装，使得每个文件中的变量天然的形成在一个闭包之中，不会污染全局变量。而浏览器端则通常是裸露的JavaScript代码片段。所以为了解决前后端一致性的问题，类库开发者需要将类库代码包装在一个闭包内。以下代码片段抽取自著名类库underscore的定义方式。
   
    (function () {
        // Establish the root object, `window` in the browser, or `global` on the server.
        var root = this;
        var _ = function (obj) {
                return new wrapper(obj);
            };
        if (typeof exports !== 'undefined') {
            if (typeof module !== 'undefined' && module.exports) {
                exports = module.exports = _;
            }
            exports._ = _;
        } else if (typeof define === 'function' && define.amd) {
            // Register as a named module with AMD.
            define('underscore', function () {
                return _;
            });
        } else {
            root['_'] = _;
        }
    }).call(this);

   首先，它通过function定义构建了一个闭包，将this作为上下文对象直接call调用，以避免内部变量污染到全局作用域。续而通过判断exports是否存在来决定将局部变量_绑定给exports，并且根据define变量是否存在，作为处理在实现了AMD规范环境（http://wiki.commonjs.org/wiki/Modules/AsynchronousDefinition） 下的使用案例。仅只当处于浏览器的环境中的时候，this指向的是全局对象（window对象），才将_变量赋在全局对象上，作为一个全局对象的方法导出，以供外部调用。

   所以在设计前后端通用的JavaScript类库时，都有着以下类似的判断：

    if (typeof exports !== "undefined") {
        exports.EventProxy = EventProxy;
        } else {
        this.EventProxy = EventProxy;
    } 
    
   即，如果exports对象存在，则将局部变量挂载在exports对象上，如果不存在，则挂载在全局对象上。

   对于更多前端的模块实现可以参考国内淘宝玉伯的seajs（http://seajs.com/），或者思科杜欢的oye（http://www.w3cgroup.com/oye/）
   
---

**参考**：

1. http://www.infoq.com/cn/articles/nodejs-module-mechanism/
2. https://liuzhichao.com/p/1669.html
