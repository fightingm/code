# 封装一个简单的express

我们利用fs,http,url等模块来完成一个简单的web服务器的封装，实现以下功能:

    1. 输入对应的路由渲染对应的页面
    2. 正确加载静态文件
    3. 支持模板引擎渲染
使用方式如下:

    const myexpress = require('./myexpress');
    const app = new myexpress();

    app.get('/', (req, res) => {
        res.send('get/');
    });
    app.get('/login', (req, res) => {
        res.render('login', {}, (err, data) => {
            res.send(data);
        });
    });
    app.post('/dologin', (req, res) => {
        res.render('dologin', {name: 'testUser'}, (err, html) => {
            res.send(html);
        })
    });

    app.listen(3000);

## 1. 开始

首先，我们需要写一个myexpress的class暴露给外部使用。大概是这个样子。

    module.exports = class myexpress{
        constructor() {
            this.init();
        }
    }
首先我们在构造函数中新增一个_get实例属性来保存get请求，然后编写一个get方法，每当app.get方法调用的时候，我们把对应的路径和处理函数保存起来，同理，post方法也是一样的：

    get(path, cb) {
        this._get[path] = cb;
    }
    post(path, cb) {
        this._post[path] = cb;
    }
然后在init函数中创建我们的web服务：

        init() {
            this.server = http.createServer((req, res) => {
                const parseUrl = url.parse(req.url, true);
                const { pathname } = parseUrl;
                const method = req.method.toLowerCase();
                // 获取到对应路由的回调函数
                const cb = this[`_${method}`][pathname];
                cb(req, res);
            });
        }
我们以'/'这个路由为例，我们看一下这里的回调执行了一个res.send方法，我们知道原生的res是没有这个方法的，所以我们需要为res添加这个方法，这个方法就是对res.writeHead和res.end方法的一个封装:

    setRes(res) {
        res.send = function(data) {
            res.writeHead(200, {"Content-Type": 'text/html; charset=utf-8'});
            res.end(data);
        }
    }
然后，不要忘了设置listen方法:

    listen(port) {
        this.server.listen(port)
    }
这时我们用node启动我们的服务，打开`localhost:3000`，就能看到页面上输出`get/`。

## 2. 静态资源文件

现在我们的服务只支持正确的路由输入，当我们的_get和_post中没有获取到对应的处理函数，我们首先要判断是不是一个静态资源文件，比如页面的css,js,image等，然后去我们的静态资源文件夹中寻找相应的文件，如果找到我们就返回文件的内容，如果没有找到我们返回404，相关代码如下

     if(cb) {
            cb(req, res);
        }else {
        // 如果没有对应的路由处理，就去静态资源目录里面找
            const extName = path.extname(pathname);
            try {
                const file = await this.loadStaticFile(pathname);
                const fileType = await this.loadContentType(extName);
                res.writeHead(200, {"Content-Type": `${fileType}; charset=utf-8`});
                res.end(file);
            } catch (err) {
                res.writeHead(404);
                res.end("404");
            }
        }
这里有一个loadStaticFile和loadContentType方法分别用来读取静态文件和根据文件后缀返回对应的Content-type,大家有兴趣可以去我的github里面看，这里不做讲解。

## 3. 模板引擎

由于个人不是很喜欢ejs的写法，所以这里选用[handlebars](http://handlebarsjs.com/)作为渲染模板。模板的语法大家去官网看文档就可以。这里不做介绍。这里res.render方法就是用来做模板渲染的方法,我们可以把这个方法同样放到setRes方法中:

    res.render = function(path, data, callback) {
        fs.readFile(`${views}/${path}.hbs`, (err, html) => {
            const renderData = handlebars.compile(html.toString())(data);
            callback(err, renderData);
        });
    }

[完整代码](https://github.com/fightingm/node_note/tree/master/%E5%B0%81%E8%A3%85%E8%B7%AF%E7%94%B1)

完。