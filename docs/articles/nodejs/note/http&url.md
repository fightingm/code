# http与url模块

## 1. http

node提供了http模块可以用来创建web服务。
该模块有一个createServer方法。
该方法传入一个函数，函数有两个参数分别是request和response,request中包含了请求的一些信息，response表示服务器的返回结果，可以使用response的writeHead方法向响应头添加字段，这里返回了状态码200，设置了Content-Type，最后调用end方法表示请求处理完毕。
该方法返回一个http服务，它有一个listen方法表示监听哪个端口。

    const http = require('http');
    http.createServer((req, res) => {
        res.writeHead(200, {"Content-Type": "text/html, charset='utf-8"});
        res.write('hello world');
        res.end();
    }).listen(8001);
这个时候打开浏览器，输入地址`localhost:8001`，就可以看到页面上输出hello world

## 2. url

node提供了url模块来处理url。
url有一个parse方法可以解析url,该方法支持两个参数，第一个参数表示要解析的url，第二个参数表示是否把get传值解析成object的形式。

     if(req.url !== '/favicon.ico') {
        const result = url.parse(req.url, true);
        const { pathname, query } = result;
        res.write(`pathname: ${pathname} `);
        res.write(`name: ${query.name} age: ${query.age}`);
    }
在上面创建的httpServer中，我们加入了对request的url的判断，过滤了favicon.ico的处理，然后将请求过来的url做了一个解析，在页面上输出请求过来的路径，以及请求的参数name和age。
这时候我们打开浏览器，输入`localhost:8001/user?name=fm&age=23`,就可以看到页面上输出`pathname: /user name: fm age: 23`

## 3. supervisor

每次修改完代码之后都要手动重启一下node服务才能生效，很麻烦，可以使用supervisor来完成node服务的自动重启。
首先全局安装supervisor

    npm i supervisor -g //npm
    yarn global add supervisor //yarn

然后命令行输入

    supervisor xxx.js 

再次修改node服务就不需要手动重启了。

完。