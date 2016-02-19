title: "Javascript后端开发学习"
date: 2016-01-26 21:51:09
categories: 
- Tech
tags: 
- javascript server
---
之前一直在用Java写后端，一直使用的是MVC模式，于是便好奇。不用Java，没有MVC，会是什么样子。考虑过Rails，只是出了学习Rails这个框架外，还需要学习诸如Ruby，Coffee之类的语言，而关键是Rails在debug模式下比较慢。所以没有什么动力。后面NodeJS出来了，然后大家开始用Javascript来写后端了。后面无意间发现了一个[NodeJS中文社区](cnodejs.org)，跟Ruby中文社区一样，还算比较活跃。于是就想着用NodeJS来写写试试，加之又想试试ECMAScript 2015，于是便有了这次尝试。<!--more-->

所有的代码都在github上，然后博客主要就是记录干了些什么事情，遇到了些什么问题，权当流水账了。也不会去将各个技术逐个从头讲起。Github repo是[forum](https://github.com/kolyjjj/forum)。

总体上的需求大概是希望能实现一个基本的论坛，用户可以注册，登录，发帖，评论。具体的需求会在每一步的时候记录。

## 架子
首先确定了NodeJS，则需要安装Node。Mac电脑，官网下载了node 4.2.6。后面还可以考虑使用[nvm](https://github.com/creationix/nvm)来管理多个node版本。装好之后就有了npm了。使用`npm init`来初始化项目：
```shell
mkdir forum
cd forum
npm init
```
使用app.js作为入口文件(npm init执行后会看到一系列问题，好好回答，在回答entry point的时候输入app.js)。

然后就想在express和koajs之前选一个。最后选择了express，想着它比较基础、稳定一些。安装express: 

```shell
npm install express --save
```
安装好了之后，根据express上的[hello world](http://expressjs.com/en/starter/hello-world.html)文档，写了app.js的内容。
```javascript
import * as Express from 'express'; 
let express = Express();

app.get('/', (req, res) => {
        res.send('Hello World!');
});

app.listen(3000, () => {
       console.log('Example app listening on port 3000!');
});
```
使用了ES6。但是这时使用`node app.js`不行，因为node还不支持ES6。这时候就需要[babeljs](https://babeljs.io/)来救场了。此处为第一次commit。

要使用babeljs，怎么用呢？直接用babel提供的[cli](https://babeljs.io/docs/setup/#babel_cli)？好像不太合适。不然找一个构建工具好了，除了用babel之外还可以做些别的事情。说到前端构建，自然就想到了[gulpjs](http://gulpjs.com/)，不为什么，只因为它比grunt要新，据说也更快。使用gulp就需要写Gulpfile，既然准备用ES6，那么gulpfile不至于用ES5来写吧。于是找了找怎么在gulpfile里使用ES6，发现也可以使用babel。整个过程大概是这样的:
* 安装gulpjs，注意必须是gulp3.9或以上。`npm install gulp -g`，`npm install gulp --save-dev`
* 安装babel相关lib。`npm install babel-core babel-preset-es2015 —save-dev`，`npm install  gulp-babel —save-dev`
* 在项目目录下创建.babelrc, 然后写入`{"presets":["es2015"]}`
* 在项目目录下创建gulpfile.babel.js，然后就可以在里面使用ES6了

babelrc的内容：
```json
{
  "presets": ["es2015"]
}
```

gulpfile.babel.js的内容：
```javascript
'use strict';

import gulp from 'gulp';
import babel from 'gulp-babel';

gulp.task('build',  () => {
  return gulp.src(['app.js', 'src/**/*.js'])
        .pipe(babel({
            presets: ['es2015']
        }))
        .pipe(gulp.dest('dist'));
});
```
这样就可以使用`gulp build`来将ES6编译成ES5了。编译之后的文件放在了dist目录下，可以使用`node dist/app.js`来启动项目。结果发现出错了，找不到express。排错后发现`import * as Express from 'express'; `有问题，应该是`import express from 'express'`。自然后面的Express也应该改成express。而`let express = Express();`改成`let app =
express();`这样就没问题了。项目启动之后，使用Chrome访问localhost:3000就可以看到输出了。除了使用浏览器之外，还可以使用[postman](https://chrome.google.com/webstore/detail/postman/fhbjgbiflinjbdggehcddcbncdddomop)这个工具来发请求，后面的post，delete等请求在手动测试的时候都使用了postman。此处为第二次提交。

使用`node dist/app.js`是可以的，只是每次代码有变动，都需要手动Ctrl+C停止，然后重新启动。这种事情不可以自动化吗？当然可以了。于是一番搜索，发现了nodemon，翻译过来就是没有demon。既然用了gulp，就一用到底，使用了[gulp nodemon](https://github.com/JacksonGariety/gulp-nodemon)。又一顿install:
``` shell
npm install —save-dev gulp-file-cache
npm install —save-dev gulp-nodemon
```
然后在gulpfile里面增加了start任务来启动app，其依赖于compile任务。compile任务是在之前的build任务基础上进行了改动，然后名字也改成了compile。所以现在的流程就是：改动代码->自动compile->改动dist里面的文件->使用nodemon跑dist里面的app.js。其中用到了file cache(文件缓存)，这样在编译ES6的时候，对于没有改动的文件就不需要编译了。此处为第三次提交。

途中，还出现了找不到module的问题。原因是使用`gulp.src(['app.js', 'src/**/*.js'])`不会保留目录结构，所以`import routers from './src/routers';`是找不到src/routers.js的，可以在dist目录里面看到app.js和routers.js在同一个目录下。一番搜索，发现`gulp.src(['app.js', 'src/**/*.js'], {base: '.'})`就可以保留目录结构了。

至此，就搭建好了基本的开发环境。有了本地的一个类似hot deploy的server，也有了一个ES6的编译机制。同时，截至目前为止，发现用得上ES6的地方也就是：
* 使用匿名函数
* 使用import以及export
* 使用const，let而不是var
后面应该还会有更多使用到的地方。

## 将routes分散到各个功能模块中去
routes全部放在app.js中觉得不太好，每个模块应该负责管理自己的routes。这样维护起来也方便一些。于是使用[express-router](http://expressjs.com/en/guide/routing.html#express-router)将routes放到了模块里。
```javascript
// app.js
import routers from './src/routers';
app.use('/api', routers);

// src/routers.js
import express from 'express';
import postRouter from './posts/router';

const router = express.Router();

router.use(function timeLog(req, res, next){
        console.log('Time: ', Date.now());
            next();
                });

router.get('/', (req, res) => {
        res.send('hello, here is the api');
            });

router.use('/posts/', postRouter); 

export default router;
```
当访问/api的时候，会由routers.js文件来处理。在routers.js文件里面，又设定了当访问/posts/的时候，由posts/router.js处理。这样就将post/路径对应的处理操作放到了src/posts文件夹下面。一个文件夹就是一个功能模块，而routers.js则扮演了一个路由分发者的角色。
还有，最后的`export default router`写成`export router`，会报错的。为什么呢？来看看export的语法:
```javascript
export { name1, name2, …, nameN };
export { variable1 as name1, variable2 as name2, …, nameN };
export let name1, name2, …, nameN; // also var
export let name1 = …, name2 = …, …, nameN; // also var, const

export default expression;
export default function (…) { … } // also class, function*
export default function name1(…) { … } // also class, function*
export { name1 as default, … };

export * from …;
export { name1, name2, …, nameN } from …;
export { import1 as name1, import2 as name2, …, nameN } from …;
```
简单来说，没有`export router`这个语法，可以写成`expore {router}`，得加上一个引号。同时，写成`export default const router = ...`也是不行的。为什么呢？`export default`后面跟着的是一个expression，就是一个可以产生值的东西，比如上面的router代表了一个值，对它进行evaluate可以得到值。如果有=号了就表示是一个statement，这个执行某些操作，其本身在javascript里面是不会产生值的。

## 数据库访问
数据总是要存到一个地方的。于是便有了数据库。由于是用node，所以第一时间想到了mongodb，同时也想试试NoSQL的感觉，所以就用了。选择了mongodb，自然要选择一个工具来同它交流。在简单的比较[MongoClient](https://github.com/mongodb/node-mongodb-native?_ga=1.95774302.336281007.1453710505)和[Mongoose](http://mongoosejs.com)之后，还是想先用一个ORM，先看看这种写法会是什么样的，后面再来对比MongoClient。
要用mongodb，首先得装一个mongodb，由于之前使用过[docker](https://www.docker.com/)，所以就直接用docker的[mongodb image](https://hub.docker.com/_/mongo/)。装好之后就可以使用mongodb了，在mac上，不能直接使用localhost，可以用docker-machine ip default来查看默认的docker machine的地址。此时其地址是192.128.99.100。
有了数据库之后，就开始安装Mongoose：
```shell
npm install mongoose
```
这里没有`--save-dev`，因为产品代码也需要mongoose。装好之后就可以使用了：
```javascript

import mongoose from 'mongoose';

const dburl = 'mongodb://192.168.99.100/test';
mongoose.connect(dburl);

const Schema = mongoose.Schema;
const blogSchema = new Schema({
      title: String, 
      author: String, 
      body: String,
      comments: [{body: String, date: Date}],
      date: {type: Date, default: Date.now},
      hidden: Boolean,
      meta: {
          votes: Number,
      favs: Number
      }
});

const Blog = mongoose.model('Blog', blogSchema);
let aBlog = new Blog({
    title: 'first blog',
    author: 'koly',
    body: 'What a beautiful world',
    comments: [{body: 'a comment', date: Date.now()}],
    hidden: false,
    meta: {
        votes: 1,
    favs: 1
    }
});
// 此处即为保存
aBlog.save(); 
```
## 将配置放到单独的文件里
上面提到本地有一个数据库的地址，是192.168.99.100。本地的是这个，但是产品环境上很可能不是这个了。对于肯定会变化的东西，自然要提前准备应对变化。方式就是使用某种机制可以方便的将可变的地址切换。于是想到首先将配置提出来，放到一个文件里面，然后通过gulp根据不同的参数将不同的配置文件打到包里。
于是，配置被放到了db_config.js里：
```javascript
// env/local/db_config.js
const db = {
  url: 'mongodb://192.168.99.100/test'
};
export default db;}
```
在使用的时候就变成：
```javascript
import dbConfig from '../../env/db_config';
mongoose.connect(dbConfig.url);
```
每个环境有自己的一个配置文件，放在自己的文件夹里。比如local就是local文件夹，配置文件就是env/local/db_config.js。在编译的时候需要根据参数将对应环境的配置打进去，具体命令是`gulp compile -env=prod`。如果没有指定，则默认是local，会使用env/local/下面的db_config.js。接收命令行参数，使用了[npm minimist](https://github.com/substack/minimist)库。
```javascript
import minimist from 'minimist';
  
const cache = new Cache();
const knownOptions = {
    string: 'env',
    default: { env: process.env.NODE_ENV || 'local' }
};

const options = minimist(process.argv.slice(2), knownOptions);
const environmentPath = `env/${options.env}/*.js`;

gulp.task('env', () => {
    return gulp.src(`${environmentPath}`)
          .pipe(cache.filter())
          .pipe(babel())
          .pipe(cache.cache())
          .pipe(gulp.dest('dist/env'));
    });

gulp.task('compile', ['env'], () => {
  ...
```
其中在给environmentPath赋值的时候使用了es6中的模板字符串。

## 接收json格式的post数据
想要在express中接收并解析json数据，搜索之后发现需要一个[body parser](https://github.com/expressjs/body-parser)。于是果断引入，安装。
```javascript
import bodyParser from 'body-parser';
app.use('/api', bodyParser.json());
```
这样发送到/api的请求都会经bodyParser解析一遍。既然是body parser，自然解析的就是请求的body了。之后通过`req.body`拿到解析之后的数据，存入数据库就好了。由于在这里没有遇到什么坑，所以就简略了。

## 对请求加一些处理条件
首先Content-Type得是application/json。这个针对post, put, patch, delete，不针对get。所以：
```javascript
// src/routers.js

router.all('*', function onlyAllowJson(req, res, next) {
  const method = req.method;
  if (lodash.includes(['GET'], method)) next();
  else {
      const contentType = req.get('Content-Type');
      if (contentType && contentType.includes('application/json')) 
         next();
      else 
      res.status(400).send('wrong Content-Type, should be Content-Type:application/json, yours is ' + contentType);
  }
});
```
其中的`lodash`是引入了[npm lodash](https://www.npmjs.com/package/lodash)。

其次，如果post请求的body是空，那么也不处理：
```javascript
// src/routers.js
router.post('*', function postShouldHasContent(req, res, next) {
  if (req.body && !lodash.isEmpty(req.body)) 
  next();
  else
  res.status(400).send('post should contain valid body');
});
```

## 创建blog的时候需要进行验证
在创建blog的时候，需要一些基本的验证。比如说blog的标题不能为空，作者不能为空，作者的名字的长度需要进行限制。这些验证操作都需要在数据存入数据库之前完成。还好，Mongoose提供了这样的验证机制。简单来说就是在定义schema的时候同时声明验证条件及验证的错误信息。
```javascript
const blogSchema = new Schema({
  title:{
    type: String,
    required: true,
    minlength: 4
  }, 
  author: String, 
  body: String,
  comments: [{body: String, date: Date}],
  date: {type: Date, default: Date.now},
  hidden: Boolean,
  meta: {
    votes: Number,
    favs: Number
  }
});
```
可以看到title的后面从String变成了一个对象，这个对象定义了title的类型和验证条件。这里required表示title必须有值，而minlength表示title的长度至少为4个字符。
其余的代码都没有变化，只是在调用的时候需要加上一些错误处理：
```javascript
  postsdb.save(req.body).then((data) => {
    res.status(200).json({id: data._id});
  }, (err)=>{
    console.log('creating post error', err);
    if ('ValidationError' === err.name)
      res.status(400).send();
    else res.status(500).json(err);
  });
```
首先要在log里面记录错误信息，然后如果是ValidationError的话，就返回400，表示bad request。
光是返回400还不行，还得有错误信息啊，不然谁知道哪里错了啊。加上title的错误信息，首先在定义Schema的时候：
```javascript
 title:{
    type: String,
    required: '{PATH} cannot be empty.', // PATH must be uppercase
    minlength: 4
  }, 
```
之前是required: true，现在变成一个字符串，表示title不能为空，如果是空的话，会返回那个字符串作为错误信息。字符串中的`{PATH}`表示当前的字段，这里就是title,这里要注意，path必须是全部大写，不然不会进行替换。同时在router里面处理一下返回的错误信息：
```javascript
 if ('ValidationError' === err.name) {
      let errorMessages = composeErrorJson(err.errors);
      res.status(400).json(errorMessages);
 }
 ```
 composeError的目的就是将错误信息重新组织一下，变成下面的样子：
 ```javascript
{
  "title": "title cannot be empty."
}
```
之后就可以增加更多的验证了：
```javascript
 author: {
    type: String,
    required: '{PATH} cannot be empty.', 
    minlength: [2, '{PATH} should be more than 2 characters.'],
    maxlength: [40, '{PATH} should be less than 40 characters.']
  }, 
  content: {
    type: String,
    required: '{PATH} cannot be empty.',
    minlength: [15, '{PATH} should be more than 15 characters.']
  },
```
## 删除blog
```javascript
 deleteOne(id) {
    return Blog.findByIdAndRemove(id);
  }
```
mongoose提供了先find然后remove的方法，并不是直接remove。然后之前判断Content-Type的地方，需要将delete方法排除掉：
```javascript
if (lodash.includes(['GET', 'DELETE'], method)) next();
```
很简单，只需要在GET后加上DELETE就行了。

## 测试
测试分为好几种，有单元测试、集成测试、功能测试等。考虑到这里只有简单的CRUD，逻辑上并不是十分复杂，所以单元测试和集成测试就不写了，只写api测试（也算是一种功能测试啦）了。api测试也有一些工具可以选用，比如[frisby](http://frisbyjs.com/)，[supertest](https://github.com/visionmedia/supertest)等。考虑到supertest可以跟[mocha](http://mochajs.org/)配合，而写单元测试我也倾向于mocha。为了风格一致，所以就选用了supertest作为api测试的工具。这里提一下supertest和[superagent](http://visionmedia.github.io/superagent/)。后者是用来发送ajax请求的，而supertest使用了superagent，所以superagent的方法在使用supertest的时候也可以用。
写测试使用mocha，跑测试也可以用mocha。当然首先就要安装了。由于使用了gulp，并且想讲测试的命令也用gulp来跑，所以选择了[gulp-mocha](https://github.com/sindresorhus/gulp-mocha)。安装：

```shell
npm install --save-dev gulp-mocha
```
使用`gulp test`来跑测试：
```javascript
gulp.task('test', ['compile'], () => {
  return gulp.src('tmp/test/**/*.spec.js', {read:false})
  // gulp-mocha needs filepaths so you can't have any plugins before it
  .pipe(mocha({reporter:'nyan'}))
  .pipe(exit());
});
```
上面的`exit()`是使用了[gulp exit](https://github.com/dreame4/gulp-exit)来退出，不然的话，跑完测试之后，server还是处于已启动的状态，不会自己关闭。这算是一个小bug，据说直接使用mocha来跑的话，不会出现这个问题。所以这算是gulp-mocha的bug。
test依赖于compile，之前的compile只编译了src下面的代码，现在需要编译test下面的了：
```javascript
 gulp.task('compile', ['env'], () => {
  
  return gulp.src(['app.js', 'src/**/*.js', 'test/**/*.js'], {base: '.'})
  .pipe(cache.filter())
  .pipe(babel({
    presets: ['es2015']
  }))
  .pipe(cache.cache())
  .pipe(gulp.dest('tmp'));
});
```
安装supertest：
```javascript
npm install supertest --save-dev
```
建立测试目录，和测试文件，然后写下第一个测试：
```javascript
import request from 'supertest';
import app from '../../app';

describe('GET /posts', ()=>{
  it('should get one post', function(done){
    request(app)
      .get('/api/posts')
      .expect('Content-Type', /json/)
      .expect(200, done);
  });
});
```
这里需要app.js里面定义的app，所以需要将它export出来：
```javascript
export default app;
```
然后使用`gulp test`就可以执行测试了。
跑成功后，就可以继续添加更多测试了，比如：
```javascript
  let aPost = {
    "title":"A new Post B",
    "author":"koly",
    "content":"Hello this is a post.",
    "comments":[],
    "hidden": false,
    "meta": {}
  };

  it('should create one post', function(done){
     request(app)
     .post('/api/posts')
     .send(aPost)
     .expect('Content-Type', /json/)
     .expect((res)=>{
      console.log('creating one post');
      res.body.should.have.property('id');
     })
     .expect(200, done);
   });
```
上面的`res.body.should.have.property('id)`使用了[shouldjs](https://github.com/shouldjs)，看起来很人性。其次，done放在最后的expect(200, done)才有效果，不然没有用，会导致测试跑不过。然后function(done)不能写成匿名函数形式，不然在babel编译之后mocha找不到done，也就没有作用。当然done放在end里面也是可以的，具体参看[代码](https://github.com/kolyjjj/forum/commit/ef3038ff450175b5713c5eee8a1aaef3b31b2320)。
还有一个问题是，测试里面各种创建blog，把数据库给污染了怎么办呢？如何做数据库回滚呢？考虑到这是api层次的测试，如果引入mongodb的任何connection会觉得不是这个层次该做的事情。api层次的事情就让api来解决。所以最后只能当作数据库里面原来就什么数据也没有，跑测试的时候创建了数据。跑完之后把数据删掉。于是有了：
```javascript
 after(function(done){
   request(app)
   .get('/api/posts')
   .expect('Content-Type', /json/)
   .end((err, res)=>{
     if (err) throw err;
     let deleteFuncs = [];
     res.body.forEach((value)=>{
       deleteFuncs.push((cb)=>{
         console.log('deleting', value._id);
         request(app)
         .delete('/api/posts/'+value._id)
         .expect(200, cb);
       });
     });
     async.series(deleteFuncs, done);
   });
 });
```
after函数会在所有测试跑完之后执行。其中使用了[asyncjs](https://github.com/caolan/async)来执行具体的删除动作。
测试就先这样了。

## 修改blog
既然有了测试，那么就尝试一下TDD(Test Driven Developmen)。
```javascript
it('should update a post', function(done){
   request(app)
   .post('/api/posts')
   .send(aPost)
   .expect('Content-Type', /json/)
   .expect(200)
   .end((err, res)=>{
     if (err) throw err;
     request(app)
     .put('/api/posts/' + res.body.id)
     .send({
      "title":"updated post",
      "author":"another author",
      "content":"updated posts content"
     })
     .expect('Content-Type', /json/)
     .expect((res)=>{
       console.log('updating a post');
       let body = res.body;
       body.title.should.be.exactly("updated post");
       body.author.should.be.exactly(aPost.author); // author cannot be udpated
       body.content.should.be.exactly("updated posts content");
     })
     .expect(200, done);
   });
 });
```
就是先创建一个post，然后去update，updata的返回时新修改的结果，author字段无法修改。
实现分为两步，第一步添加路由，第二步数据库操作：
```javascript
router.put('/:id', (req, res)=>{
  console.log('request body for updating post', req.body);
  postsdb.update(req.params.id, req.body).then((data)=>{
    createResponseWhenPostNotFound(data, req.params.id, res, (data)=>{
      res.status(200).json(data);
    });
  }, (err)=>{
    if ('ValidationError' === err.name) {
      let errorMessages = composeErrorJson(err.errors);
      res.status(400).json(errorMessages);
    }
    else res.status(500).json(err);
  });
});
```
```javascript
update(id, data) {
   return Blog.findByIdAndUpdate(id, {
     title: data.title,
     content: data.content
   }, {
     new: true,
     runValidators: true
   });
 },
 ```
 这里findByIdAndUpdate的第三个参数是options，其中new为true表示返回新的记录，否则返回的是老记录；runValidators为true表示修改的时候需要进行验证。这两个默认都为false。

 到这里就完成了blog的CRUD操作，后面可能先作一些修改，比如增加last_edit_time属性等。还有就是comment的CRUD，然后就是用户及权限的处理。

 此为一。
