title: Javascript后端开发(二)
categories:
- Tech
tags: 
- NodeJS
---

本文接着上文[《Javascript后端开发学习》](http://koly.me/2016/01/26/Javascript%E5%90%8E%E7%AB%AF%E5%BC%80%E5%8F%91%E5%AD%A6%E4%B9%A0/)。将继续记录学习NodeJS后端开发的“流水”。这篇主要是记录comments的CRUD。<!-- more -->
## 编码前的思考
主要有两个方面：
* 数据库的表设计，即post和comment的关系
* api的设计，即url的设计

先说第一个，表设计。详细的表设计流程可以参考[《数据库设计基础》](http://koly.me/2015/11/03/%E3%80%90%E8%AF%91%E3%80%91%E6%95%B0%E6%8D%AE%E5%BA%93%E8%AE%BE%E8%AE%A1%E5%9F%BA%E7%A1%80/)。这里明显有两个实体，一个是post，另一个是comment。实体之间的关系一般有一对一，一对多，多对多。由于一个post可以对应多个comment，而同一个comment不能同时属于两个不同的post，所以post和comment的关系是一对多关系，即一个post对应多个comment。如果在关系数据库中，则需要建立一张post表，一张comment表，然后在comment表中使用一列来存储post的id以表示该条记录所属于的post。在mongodb中对一对多的实现由两种方式:
* 嵌入式文档([embedded document](https://docs.mongodb.org/manual/tutorial/model-embedded-one-to-many-relationships-between-documents/))
* 文档引用([document references](https://docs.mongodb.org/manual/tutorial/model-referenced-one-to-many-relationships-between-documents/))。

嵌入式文档的意思就是说在表示post的一个json里面有一个数组，然后这个数组的每个元素就是一个comment。文档引用就是，post是一个文档，而comment是另一个文档，post文档有一个`_id`，在comment文档里面增加一个属性，然后使用post的那个`_id`来表示该comment属于哪个文档。第一种方式在读取post的时候就可以读取该post对应的comments，对于一个post，只需要一次查询就可以得到该post的内容和对应的所有comments。第二种方式需要两次查询才能拿到post及对应的comments。这里考虑到从用户的角度来说post相对comment来说更为重要一些，有的时候甚至没有comment也没有太大的影响。同时如果comments过多的话可能会影响响应时间，而用户可能希望首先看到post的内容。所以从用户的行为习惯出发，更倾向于将post和comment分开，使用分开的查询，这样可以尽可能保证post的相应速度，优先显示出post。技术上来说，使用一个单独的请求来获取comment，在comment变多的时候可以考虑分页处理，而不会影响post，甚至于将comment作为单独的一个模块独立出去，比如[DISQUS](https://disqus.com/home/explore/)。所以最后决定采取第二种方式。

api的设计，就遵循Restful API的设计风格，考虑到两个资源的关系，comment的主api是`/api/post/<post_id>/comments/`，对应的CRUD就是POST, GET, PUT, DELETE。

那么想到这里就可以试着开始开发了。

## 自顶向下
所谓的“自顶向下”就是从api开始写，知道写到最后的数据库操作。这里顶就是上层，下就是底层。一般认为数据库比较底层。从api开始写，自然就是api的定义，这属于路由(router)的一部分。按照Restful API的风格，comments的路由应该是：
```javascript
create => POST /api/posts/<post_id>/comments
get comments => GET /api/posts/<post_id>/comments
get one comment => GET /api/posts/<post_id>/comments/<comment_id>
update comment => PUT /api/posts/<post_id>/comments/<comment_id>
delete comment => DELETE /api/posts/<post_id>/comments/<comment_id>
```
从url中可以看到，comments与posts是一种从属关系，既comments是属于posts的。在开发的时候，comments被当做一个单独的独立于comments的模块，主要是基于其本身可以可能独立出去的性质。所以comments在一个单独的文件夹下面。先是总的路由分发：
```javascript
// routers.js
import commentRouter from './comments/router';
...
router.use('/posts/:id/comments/', commentRouter);
```
然后是comments模块自身的路由，刚开始只实现了一个假的get comments:
```javascript
// src/comments/
'use strict';

import express from 'express';
import commentsdb from './commentsdb';

const router = express.Router();

router.get('/', (req, res)=>{
  res.status(200).send('hello, comments');
});

export default router;
```
这样就将comments相关的路由分配到了对应的comments模块。同样也建立`commentsdb.js`来处理数据库的相关操作，当然一开始里面也是什么也没有，只作一个架子。
## 数据操作代码的重构
在做comments的数据库的记录插入的时候，发现也需要创建一个数据库连接以及建立一个model，而这部分代码跟posts那边的代码几乎一样，对于重复代码，自然要重构掉。于是在开发新功能前先进行了重构：
```javascript
// src/database/index.js

import mongoose from 'mongoose';
import dbConfig from '../../env/db_config';

mongoose.connect(dbConfig.url);

const db = mongoose.connection;
db.on('error', console.error.bind(console, 'connection error:'));
db.once('open', ()=>{
  console.log('connected to mongodb');
});

const Schema = mongoose.Schema;

function createModel(name, schema) {
  return mongoose.model(name, new Schema(schema));
}

export default createModel;
```
将重复代码提取到了`database/index.js`中，将`createModel`方法export出去，作为提供数据操作的接口。到这儿的时候，心里想这个`index.js`文件会被多次`import`，这种多次import会不会导致代码的多次执行，这样就会有多个connection。后来google了一下，发现只会执行一次，换句话说你export出去的东西应该都是一个单例的Object。
```javascript
// src/posts/postsdb.js
import createModel from '../database/index';
const schema = {
  title:{
    type: String,
    required: '{PATH} cannot be empty.', // PATH must be uppercase
    minlength: [4, '{PATH} should be more than 4 characters.']
  }, 
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
  comments: [{body: String, date: Date}],
  created_date: {type: Date, default: Date.now},
  last_edit_date: {type: Date, default: Date.now},
  hidden: Boolean,
  meta: {
    votes: Number,
    favs: Number
  }
};

const Blog = createModel('Blog', schema);
```
对比之前的代码，postsdb.js中剥离出了具体的数据库相关操作。在postsdb中的代码就是完全关于posts的数据库操作的。这里就实现了SRP(Single Responsiblity Principle)，即单一职责原则，此时的postsdb相比之前，就做了一件事情，之前既要负责数据库连接的建立也要负责posts的CRUD。
## 创建comment
先来一段测试代码：
```javascript
// test/comments/api.spec.js
'use strict';

import request from 'supertest';
import should from 'should';
import async from 'async';
import app from '../../app';

describe('comments operation', ()=>{
  let aPost = {
    "title":"A new Post for comment test",
    "author":"koly",
    "content":"This is a post for testing comments",
    "hidden":false,
    "meta":[]
  };
  const aComment =  {
    "author":"koly",
    "content":"this is a comment"
  };

  it('should create a comment', function(done){
    request(app)
    .post('/api/posts')
    .send(aPost)
    .expect(200)
    .end((err, res)=>{
      if (err) throw err;
      console.log('post id', res.body.id);
      request(app)
      .post('/api/posts/' + res.body.id + '/comments')
      .send(aComment)
      .expect('Content-Type', /json/)
      .expect((res)=>{
        res.body.should.have.property('_id');
      })
      .expect(200, done);
    });
  });
});
```
可以看到这里跟posts的测试代码也有一些重复的地方，上面一节说了对于重复的地方要进行重构。但是这里笔者以为对于产品代码和测试代码，重构的标准应当进行区分。产品代码更在乎简洁性，而测试代码对简洁性的要求不高，更在乎的是可读性，以及测试的阅读完整性。比如，这个测试在这里，只需要看这个单独的文件就能够看懂，而如果将重复的部分抽取到其他地方的话，那么在看这个测试的时候，就需要跳转到其他文件，先理解了其他文件之后，才能继续回来阅读，这样就增加了多余的跳转，同时割裂了阅读的流畅性。
接下来就是实现:
```javascript
// src/comments/router.js
'use strict';

import express from 'express';
import commentsdb from './commentsdb';

const router = express.Router({mergeParams: true});

router.get('/', (req, res)=>{
  res.status(200).send('hello, comments');
});

router.post('/', (req, res)=>{
  console.log('request params', req.params);
  let aComment = Object.assign({postId:req.params.id}, req.body); 
  commentsdb.save(aComment).then((data)=>{
    console.log('creating comment successfully', data);
    res.status(200).send(data);
  }, (err)=>{
    console.log('fail to create comment', err);
    res.status(400).send();
  });
});

export default router;
```
这里有个地方需要注意`express.Router({mergeParams: true})`，之前在routers.js中的`router.use('/posts/:id/comments/', commentRouter);`可以让对comments的请求到达comments模块，但是到了之后`/posts/:id/comments`却无法通过`req.params`得到，后面google了一下需要加上`mergeParams`参数才能够得到上一层的url中的参数。其余代码这里就不贴了。
## 获取comments
获取comments在这里就只有一个需求，就是列出属于单个post的所有comments：
```javascript
router.get('/', (req, res)=>{
  commentsdb.getAll(req.params.id).then((data)=>{
    res.status(200).json(data);
  }, (err)=>{
    console.log('failed to get all comments', err);
    res.status(500).send();
  });
});
```
这里，在测试的`before`方法中实现了对post的创建，这样在其余测试中就可以部分省略创建post：
```javascript
// test/comments/api.spec.js

  let postId;

  before(function(done){
    request(app)
    .post('/api/posts')
    .send(aPost)
    .expect((res)=>{
      postId = res.body.id;
    })
    .expect(200, done);
  });
```
## 增加独立的router来处理404的情况
先贴一段创建comment的代码：
```javascript
// src/comments/router.js
router.post('/', (req, res, next)=>{
  let aComment = Object.assign({postId:req.params.id}, req.body); 
  commentsdb.save(aComment).then((data)=>{
    if (lodash.isEmpty(data)) return next();
    console.log('sending 200');
    res.status(200).send(data);
  }, (err)=>{
    console.log('fail to create comment', err);
    res.status(400).send();
  });
});
```
由于comments属于post，所以在创建comment的时候需要传入postID，以确定该comment到底属于谁。既然传入了postID，那么自然存在postID找不到的情况，这时就应该返回404错误。对于404错误的返回，之前在post的router中使用的是`res.status(404).send();`，这里换一种方式。上面的代码中，如果返回的data是空，那么直接`return next()`，`next()`表示让后面的router函数来处理，而`return`表示跳过后面的操作，后面的函数为：
```javascript
// src/routers.js

router.post('*', (req, res, next)=>{
  console.log('inceptor request', req.method, req.path);
  res.status(404).send();
});
```
这里在router分发的地方做一些全局的操作，这里将404的返回放到了所有router函数之后。还有就是回过头去想，在创建comments的时候并不需要判断data来返回是否404，因为就算传入的是错误的postId，已然会创建成功，所以后面做了一些改动。
## 关系带来的考量
从Restful的角度来看，万物皆资源。posts是一种资源，comments也是一种资源。这两种资源并不是独立的，他们具有一种“属于”关系，也就是一对多关系。那么在对这两种资源进行操作的时候，这种“属于”关系会带来什么影响呢？
一般来说，资源的操作都包括四种，即增、删、改、查。我们分别考虑一下posts的操作对comments的影响以及comments的操作对posts的影响。
操作posts：
* create，posts可以没有comments，所以对post的创建可以跟comments没有关系
* update，对posts进行修改的时候，一般情况下可以独立进行。这里的一般情况是指所改动的地方与comments没有关系。比如posts跟comments的关系是由postId来维系，那么修改post的时候没有修改postId，就没有必要对comments进行修改
* get posts，查看posts列表的时候，一般不需要显示comments，所以跟comments没有关系；但如果需要显示comments的数量，就跟comments有关了
* get post，即获取单独的一个post的时候，一般来说需要获取该post对应的comments，因此跟comments有关
* delete post，删除post的时候需要将该post下面的comments全部删除

操作comments：
* create，由于comments从属于某一个post，所以在创建comments的时候，需要指定一个postId，并且还需要验证该postId是有效的
* update comments，如果不更改comment的从属关系，那么只需要更改comment，不太需要考虑post
* get comments，从属关系，所以需要一个postId，并且需要验证该postId的有效性
* get comment, 需要一个postId，或者一个单独的commentId
* delete comment，删除comment的时候，一个commentId就足够了。

这里考量的目的主要有两个：
* 遇到资源与资源之间的关系的时候，代码就开始变复杂了。关系越来越纷杂的时候，代码就更复杂了。所以在关系出现或者增加的时候，需要考虑系统的设计或重构。
* 关系所带来的影响可以举一反三，一对多带来的影响可以进行具体的泛化
这里具体一点，就是在刚刚实现创建comment的时候，并没有验证postId的有效性，补上：
```javascript
// src/comments/router.js
router.post('/', (req, res, next)=>{
  postsdb.getOne(req.params.id)
  .then((data)=>{
    if (lodash.isEmpty(data)) return next();

    let aComment = Object.assign({postId:req.params.id}, req.body); 
    commentsdb.save(aComment).then((data)=>{
      if (lodash.isEmpty(data)) return next();
      res.status(200).send(data);
    }, (err)=>{
      console.log('fail to create comment', err);
      res.status(400).send();
    });
  }, (err)=>{
    res.status(404).json({message: 'cannot find post id.'});
  });
});
````
这样，comments的建立就变得复杂了一些，后面我们再来看看能不能进行简化，以及如何简化。


