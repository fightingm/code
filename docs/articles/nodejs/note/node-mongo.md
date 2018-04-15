# node操作mongodb

## 开始

首先安装mongodb,然后在终端启动mongodb。这里使用的mongodb版本是3.6.3。

    sudo mongod

然后初始化nodeweb服务,这里使用[上篇文章](/articles/nodejs/note/myexpress)封装的myexpress。

    const myexpress = require('./myexpress');
    const app = new myexpress();

    app.listen(3000);

然后引入MongoClient模块，并定义要连接的数据库地址。

    const { MongoClient } = require('mongodb');
    const dburl = "mongodb://localhost:27017/";



## 创建数据库，集合

首先，我们创建一个名为test的数据库:

    MongoClient.connect(dburl+'test', function(err, db) {
        if (err) throw err;
        console.log("数据库已创建!");
    });
紧接着，还是在这个回调函数里面，我们创建一个user的集合:

    const dbase = db.db("test");
    dbase.createCollection('user', function (err, res) {
        if (err) throw err;
        console.log("创建集合!");
        db.close();
    });
这样我么就有了数据库和集合，接下来我们就可以向user集合里面增加，删除，修改，查询数据了。

## 查询数据

这里使用find方法查询到user集合下面的所有数据，把它转换成数组，然后传入模板文件中渲染:

    app.get('/', (req, res) => {
        MongoClient.connect(dburl, function(err, db) {
            if (err) throw err;
            const dbo = db.db("test");
            dbo.collection("user").find().toArray((err, result) => {
                if(err) {
                    console.log(err);
                    return;
                }
                res.render('users', {users: result}, (error, html) => {
                    res.send(html);
                });
                db.close();
            })
        });
    });
然后在我们的模板文件夹下新建users.hbs用来展示查询到的数据:

    {{#each users}}
        <p>{{name}}, {{age}}</p>
    {{/each}}
现在应该看不到任何数据，接下来我们添加几条数据。

## 增加数据

增加数据的方法有insert,insertOne和insertMany,这里我们增加两条数据，使用insertMany方法:

    app.get('/add', (req, res) => {
        MongoClient.connect(dburl, function(err, db) {
            if (err) throw err;
            const dbo = db.db("test");
            const users = [
                {"name": "dxx", "age": 22},
                {"name": "xkm", "age": 22}
            ];
            dbo.collection("user").insertMany(users, (err, result) => {
                if(err) {
                    console.log("数据插入失败");
                    return;
                }
                res.send('添加成功');
                db.close();
            });
        });
    });
我们首先访问路由`localhost:3000/add`，然后页面上显示添加成功之后，回到`localhost:3000/`,就能看到我们新增的数据了。

## 修改数据

这里我们使用updateOne方法将xkm的age修改成23:

    app.get('/update', (req, res) => {
        MongoClient.connect(dburl, function(err, db) {
            if (err) throw err;
            const dbo = db.db("test");
            const oneUser = {"name": "xkm"};
            const updateObj = {$set: {"age": 23}};
            dbo.collection("user").updateOne(oneUser, updateObj, (err, result) => {
                if(err) {
                    console.log(err);
                    return;
                }
                res.send('修改成功');
                db.close();
            });
        });
    });
先访问`localhost:3000/update`，然后页面上显示修改成功之后，回到`localhost:3000/`,就能看到我们修改后的的数据了。

## 删除数据

这里使用deleteOne方法删除name为xkm的数据:

    app.get('/del', (req, res) => {
        MongoClient.connect(dburl, function(err, db) {
            if (err) throw err;
            const dbo = db.db("test");
            const oneUser = {"name": "xkm"}
            dbo.collection("user").deleteOne(oneUser, (err, result) => {
                if(err) {
                    console.log(err);
                    return;
                }
                res.send('删除成功');
                db.close();
            });
        });
    });
先访问`localhost:3000/del`，然后页面上显示删除成功之后，回到`localhost:3000/`,就能看到我们删除后的的数据了。

[完整代码](https://github.com/fightingm/node_note/tree/master/mongodb)

完。