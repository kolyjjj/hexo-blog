title: "访问vagrant box中的mongodb出错"
date: 2015-09-20 14:23:53
categories: 
- Tech
tags: 
- vagrant
---

事情是这样子的：我想学习一下nodejs和mongodb，但是又不想在本机上安装一个mongodb，所以我就弄了一个ubuntu的vagrant box（64位的哦），然后把mongodb安装到了这个box里面。结果在使用代码访问这个box里面的mongodb的时候，却出错了...<!--more-->

### \# 错误情况
出错条件非常简单：
* 有一个ubuntu 64 bit的vagrant box
* 在这个box里面安装一个mongodb，运行mongodb(sudo service mongod start)
* 把mongodb的端口映射到host的端口(config.vm.network "forwarded_port", guest: 27017, host: 10087)，然后reload box哦
* 在外面的host主机里面使用nodejs代码访问box里面的mongodb

错误消息：
```
node_modules/mongodb/lib/server.js:235
        process.nextTick(function() { throw err; })
mongoError: server localhost:10087 received an error {"name":"MongoError","message":"write EPIPE"}
```
刚开始google了一下，关键字是“mongodb error write EPIPE”，结果出来的东西完全不是我想要的。后来突然想了想，会不会是node的错误，而不是mongo的错误呢。于是换成“node error write EPIPE”，然后出来的结果就比较中意了。大意就是说对方中断了连接(connection)。这下知道是vagrant中的mongo问题。于是再次更换搜索词，变成“vagrant mongdo”，运气比较好，刚好有一篇文章，[这里](http://randomgeekery.org/tools/2014/08/06_mongo-vagrant-connect.html)。参考这篇文章，就把问题给解决了。
中途为了确认是不是mongodb有没有连接上，在代码里面更改了mongo url，结果出现无法连接的错误，证明mongodb是连接上了的。恩，还是首先要确定问题的范围。
这里贴上连接的代码，使用了koajs：
```
var koa = require('koa');
var mongo = require('koa-mongo');
var assert = require('assert');
var url = 'mongodb://localhost:10087/test';

var app = koa();

app.use(mongo({
  uri: 'mongodb://localhost:10087/test', //or url
  max: 100,
  min: 1,
  timeout: 30000,
  log: false
}));
```

### \# 原因
原因就是在`/etc/mongod.conf`中有一行：
```
# Listen to local interface only. Comment out to listen on all interfaces.
bind_ip = 127.0.0.1

```
这行的意思是说，我这个mongodb只接受本地(127.0.0.1)的连接，别的么，呵呵，我就关掉了。

### \# 解决方案
把上面configuration里的那行注释掉就好了：

```
# Listen to local interface only. Comment out to listen on all interfaces.
# bind_ip = 127.0.0.1

```
行首加一个`#`。

-----
总结一下，遇到问题，还是要冷静分析，首先尽可能缩小问题的范围。然后再寻找解决方案。google的搜索词必要时候多换几个试试。最后，这种类似的有警醒意义的东西自然要用博客的方式记录下来。
