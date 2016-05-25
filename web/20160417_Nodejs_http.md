NodeJS原生http模块提供了创建http请求和服务的基本功能。

官网Api接口地址：https://nodejs.org/api/http.html

## 接口

1. Class:http.Agent
2. Class:http.ClientRequest
3. Class:http.Server
    * server.listen(handle[, callback])
    * server.listen(path[, callback])
    * server.listen(port[, hostname][, backlog][, callback])
    * server.setTimeout(msecs, callback)
4. Class:http.ServerResponse
    * response.getHeader(name)
    * response.sendDate
    * response.setHeader(name, value)
    * response.statusCode
    * response.statusMessage
    * response.write(chunk[, encoding][, callback])
    * response.writeHead(statusCode[, statusMessage][, headers])
5. Class:http.IncomingMessage
6. http.METHODS
7. http.STATUS_CODES
8. http.createClient([port][, host])
9. http.createServer([requestListener])
10. http.get(options[, callback])
11. http.globalAgent
12. http.request(options[, callback])

## 示例

**处理Get请求**

    var http = require('http');

    http.createServer(function (request, response){
      response.writeHead(200, {'Content-Type': 'text/plain'});
      response.end('Hello World\n');
    }).listen(8080, "127.0.0.1");

    console.log('Server running on port 8080.');

**处理POST请求**

    var http = require('http');

    http.createServer(function (req, res) {
        var content = "";

      req.on('data', function (chunk) {
        content += chunk;
      });

      req.on('end', function () {
        res.writeHead(200, {"Content-Type": "text/plain"});
        res.write("You've sent: " + content);
        res.end();
      });

    }).listen(8080);

**发出request请求**

    var postData = querystring.stringify({
        'msg' : 'Hello World!'
    });

    var options = {
      hostname: 'www.google.com',
      port: 80,
      path: '/upload',
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Content-Length': postData.length
      }
    };

    var req = http.request(options, function(res) {
      console.log('STATUS: ' + res.statusCode);
      console.log('HEADERS: ' + JSON.stringify(res.headers));
      res.setEncoding('utf8');
      res.on('data', function (chunk) {
        console.log('BODY: ' + chunk);
      });
    });

    req.on('error', function(e) {
      console.log('problem with request: ' + e.message);
    });

    // write data to request body
    req.write(postData);
    req.end();

## 参考

1. http://javascript.ruanyifeng.com/nodejs/http.html