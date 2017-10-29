# Express

* 基于node.js，快速、开放、极简的web开发框架
* `npm install express --save`安装express

* 使用`npm install nodemon -g`来使node热更新🆕

## 其他特性

* `app.get、app.post`分别开发`get`和`post`接口
* `app.use`使用模块，在主文件中引用子模块
* `res.send`、`res.json`、`res.sendfile`响应不同的内容（返回文本、返回JSON、返回文件📃）

## MongoDB+Mongoose
### 非关系型数据库

* 安装好`mongoDB`
* 启动 `mongod --config /usr/local/etc/mongod.conf`
* `npm install mongoose -save`安装mongoose
* 通过`mongoose`操作`mongodb`，存储的就是`json`，相对`MySQL`来说，要容易很多

## Mongoose基础使用

* `Connect`链接数据库链接
* 定义文档模型，`Schema`和`model`新建模型
* 代一个数据库文档对应一个模型，通过模型对数据库进行操作
* `String、Number`等数据结构
* `create、remove、update`分别用来增、删、改的操作
* find和findOne用来查询数据


```JavaScript
const express = require('express')
const mongoose = require('mongoose')
//链接mongodeb
const DB_URL = 'mongodb://127.0.0.1:27017/test'
mongoose.connect(DB_URL)
mongoose.connection.on('connected',function () {
  console.log('mongo connnect success');
})
//类似于mysql的表 mongo里有文档、字段的概念
const User = mongoose.model('user', new mongoose.Schema({
  user:{type:String,require:true},
  age:{type:Number,require:true}
}))
//新增数据
// User.create({
//   user:'zhangjianan',
//   age:21
// },function (err,doc) {
//   if (!err) {
//     console.log(doc);
//   }else{
//     console.log(err);
//   }
// })
//删除数据
// User.remove({'age':21},function (eer,doc) {
//   console.log(doc);
// })
//更新数据
// User.update({'user':xiaoming},{'$set':{age:26}},function(){err.doc}{
//   console.log(doc);
// })
//新建App
const app = express()

app.get('/',function (req,res) {
  res.send('<h1>hello world</h1>');
})

app.get('/data',function (req,res) {
  User.find({},function (err,doc) {
    res.json(doc)
  })
})

app.listen(9093,function functionName() {
  console.log('Node app start at port 9093');
})
```

## Express
