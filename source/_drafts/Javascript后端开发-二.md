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
对比之前的代码，postsdb.js中剥离出了具体的数据库相关操作。在postsdb中的代码就是完全关于posts的数据库操作的。这里就实现了**SRP(Single Responsiblity Principle)**，即单一职责原则，此时的postsdb相比之前，就做了一件事情，之前既要负责数据库连接的建立也要负责posts的CRUD。
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
可以看到这里跟posts的测试代码也有一些重复的地方，上面一节说了对于重复的地方要进行重构。但是这里笔者以为对于产品代码和测试代码，重构的标准应当进行区分。产品代码更在乎简洁性，而测试代码对简洁性的要求不高，更在乎的是可读性，以及测试的**阅读完整性**。比如，这个测试在这里，只需要看这个单独的文件就能够看懂，而如果将重复的部分抽取到其他地方的话，那么在看这个测试的时候，就需要跳转到其他文件，先理解了其他文件之后，才能继续回来阅读，这样就增加了多余的跳转，同时割裂了阅读的流畅性。
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

## 更新comment
这里跟更新post类似，使用put方法，只是需要对传进来的post id进行存在性验证。看看关键代码：
```javascript
// src/comments/router.js

router.put('/:commentId', (req, res, next) => {
 postsdb.getOne(req.params.id)
 .then(data => {
   if (lodash.isEmpty(data)) return next();

   let newComment = req.body;
   commentsdb.update(req.params.commentId, newComment)
   .then(data=>{
     res.status(200).send(data);
   }, err => {
     console.log('fail to update comment', err);
     res.status(400).send();
   });
 }, err => {
   console.log('updating comments ' + req.params.commentId + ' error with post id ' + req.params.id);
   res.status(404).json({message: 'cannot find post id.'});
 });
});

// src/comments/commentsdb.js

  update(id, data) {
    console.log('commentsdb updating', id, data);
    return Comment.findByIdAndUpdate(id, {
      author: data.author,
      content: data.content,
      last_edit_date: Date.now()
    }, {
      new: true,
      runValidator: true
    });
  },

```
这里的代码跟之前创建comment的代码有些类似，同时看起来不是很爽，使用promise来表达同步思维，仍然让人觉得不是很舒服。下面就来重构重构。
## co的引入
[co](https://github.com/tj/co)，是一个库，可以借助ES6的[generator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators)，关于generator的解释可以参考我的博客《ES6 generator探索》(http://koly.me/2015/07/25/ES6-generator%E6%8E%A2%E7%B4%A2/)，这里就不讲generator了，直接看看重构之前和之后的代码：
重构前：
```javascript
router.post('/', (req, res, next)=>{
  postsdb.getOne(req.params.id)
  .then((data)=>{
    if (lodash.isEmpty(data)) return next();

    let aComment = Object.assign({postId:req.params.id}, req.body); 
    commentsdb.save(aComment).then((data)=>{
      res.status(200).send(data);
    }, (err)=>{
      console.log('fail to create comment', err);
      res.status(400).send();
    });
  }, (err)=>{
    res.status(404).json({message: 'cannot find post id.'});
  });
});
```

重构后：
```javascript
// src/comments/router.js
import co from 'co';
router.post('/', (req, res, next)=>{
  co(function *(){
    let aPost = yield postsdb.getOne(req.params.id);
    if (lodash.isEmpty(aPost)) return next();
    let aComment = Object.assign({postId:req.params.id}, req.body); 
    let result = yield commentsdb.save(aComment);
    res.status(200).send(result);
  })
  .catch(err => {
    console.log('err', err);
    if (err.kind === 'ObjectId') return res.status(404).json({message: 'cannot find post id'});
    res.status(400).json({message:'valiation error'});
  });
});
```
可以看到重构之后的代码比原来的代码可读性要好一些了，更“同步”了。在实现的过程中，需要引入[co](https://github.com/tj/co)，同样也是通过npm安装，然后为了让babeljs可以编译generator，还需要安装[babel-polyfill](https://babeljs.io/docs/usage/polyfill/)。并且在`app.js`中添加`import "babel-polyfill";`

## 使用ES7中的async及await
上面的co及generator的形式已经不错了，但为了追赶潮流，我们还可以使用ES7中的async及await:
```javascript
// src/comments/router.js

import {wrap} from '../utils/utils';

router.post('/', wrap(async function (req, res, next) {
    try{
      let aPost = await postsdb.getOne(req.params.id);
      if (lodash.isEmpty(aPost)) return next();
      let aComment = Object.assign({postId:req.params.id}, req.body); 
      let result = await commentsdb.save(aComment);
      res.status(200).json(result);
    } catch (err){
      console.log('err', err);
      if (err.kind === 'ObjectId') return res.status(404).json({message: 'cannot find post id'});
      res.status(400).json({message:'valiation error'});
    }
}));
```
其中的wrap是由于express本身不支持async而添加的，其代码：
```javascript
// src/utils/utils.js

'use strict';

const wrap = fn => (...args) => fn(...args);

export {wrap};
```
wrap函数接收一个函数作为参数返回另一个函数。返回的函数接收多个参数，并调用wrap函数接收的参数同时应用这些参数。async function也是一个function，所以其内部代码的执行，已然需要对该函数的调用。

为了使babeljs能够编译async，需要引入[babel-plugin-transform-async-to-generator](https://www.npmjs.com/package/babel-plugin-transform-async-to-generator)，并且在`.babelrc`中加上一些代码：
```javascript
{
    "presets": ["es2015"],
    "plugins": ["transform-async-to-generator"]
}
```
## 删除comment
继续使用上面的async进行删除：
```javascript
// src/comments/router.js

router.delete('/:commentId', wrap(async function (req, res, next) {
 try {
   await commentsdb.deleteOne(req.params.commentId);
   res.status(200).send();
 } catch (err) {
   console.log('delete comment error', err);
   res.status(404).json({message: 'comment not found'});
 }
}));
```
注意这里并没有对传入的post id进行存在验证。我觉得没有必要进行验证，删除comment的时候对post没有影响。这时候有个问题，既然传入的post id没有作用，那么有必要进行收集吗？就delete操作本身来看，没有必要，但是从comment作为资源的角度看，依然有必要，因为url代表了一种关系，删除的不是一个独立的comment，而是一个跟post有关系的comment。纵然没有用到post id，但关系还是需要url来表示的。
## 删除post
根据之前的post与comment的关系分析，在删除post的时候，需要同时删除属于该post的comments。这里就设计到“事务”，即删除comments是删除post的一部分，如果comments删除失败，而post删除成功，那么整体上就不算成功。关于事务，最经典的例子就是简单的银行转账的例子，A账户需要把钱转到B账户，就有两个步骤：1）扣除A账户上的钱；2）将扣除的钱加到B账户上。处理转账操作时需要将这两个操作捆绑在一起，必须两个都成功，转账操作才算成功，只要其中任意一个失败，转账就失败。失败的情况就有两种第一步扣除失败及第二部增加失败。扣除失败的情况简单，只需要返回转账失败就可以了；而增加失败的情况多出一步，除了返回转账失败外，还需要将A账户上扣除的钱增加回去(这一步称为回滚)。那么这两步捆绑的操作，就是一个“事务”。
如何实现事务呢？经过一番搜索，发现mongodb本身不支持事务处理，如果要实现事务操作的话，需要自己实现诸如[两步提交]()的形式。心里一想，这也有点复杂了吧，这又不是一个分布式系统。mongodb为什么不支持事务处理呢？有几个原因，其中一个是部分系统并不需要事务，而实现事务会影响数据库的可扩展性。那么回到post和comments，这里到底需要不需要严格的事务呢？首先post对于用户来说是重要的，但是comment相对而言就不太重要了。一个post的comment少了一条或者几条，应该并没有多大的影响。而且如果一个post要被删除了，意味着该post存在的意义不大了，既然post存在的意义不大，那么可以认为其comments的意义就不大。所以这里post和comment的权重是不一样的，post的权重要大一点，起决定性作用，即post的删除成功是由post决定的。所以删除成功的情况，必须保证post确实被删除了。删除成功比较好办，两个都删除了就返回成功。如果post删除失败或者comments删除失败，就返回失败。只是由于这里没有事务操作，所以没有回滚。这时候就有两种处理顺序：1)删除comments，成功后删除post；2）删除post，成功后删除comments。第二种情况，如果post删除的时候失败了，用户无法通过同样的接口重试，这样数据库里面就会存在一些没有对应post的comments。从用户的角度，删除失败之后再去访问post应该可以可以访问，但是实际确无法访问；第一种情况，删除失败，用户依然可以访问post，并且可以重试。明显，这里选择第二种处理方式。
## 异常情况处理

