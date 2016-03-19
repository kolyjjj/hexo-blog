title: Javascript后端开发(三)
categories:
- Tech
tags:
- Nodejs
---
接上面一篇，这篇会记录用户的创建以及基本的验证(authentication)和授权(authorisation)。当然跟之前一样，给出的代码不一定是完全的，只是为了说明，完全的代码请到github的[forum](https://github.com/kolyjjj/forum)查看。<!-- more -->

## Users的CRUD
由于前面已经分别记录过posts和comments的CRUD了。所以这次就放一起记录了。刚开始当然是创建测试：
```javascript
// test/users/api.spec.js
'use strict';

import request from 'supertest';
import should from 'should';
import app from '../../app';

describe.only('users api', ()=>{
  let anUser = {
    "name": "koly",
    "accountId": "koly",
    "password":"123456",
    "email":"kolyjjj@163.com",
    "mobile": "12345678901"
  };

  it('should create a new user', function(done){
    request(app)
    .post('/api/users/')
    .send(anUser)
    .expect('Content-Type', /json/)
    .expect(200)
    .end((err, res)=>{
      if (err) throw err;
      res.body.should.have.property('id') ;
      done();
    });
  });
});
```

然后是添加路由：
```javascript
router.use('/users/', userRouter);
```

独立的users的api和usersdb：
```javascript
// src/users/router.js

'use strict';

import express from 'express';
import lodash from 'lodash';
import usersdb from './usersdb';
import {wrap} from '../utils/utils';
import {NotFound} from '../errors/errors';

const router = express.Router();

router.post('/', wrap(async function(req, res, next){
  console.log('creating user', req.body);
  res.status(200).json('hello world');
}));

export default router;

// src/users/usersdb.js

'use strict';

const usersdb = {};

export default usersdb;
```
这样就把users的架子搭起来了。

### 创建默认增删改查
在实现users的增删改查的时候发现，跟之前posts和comments的一些操作差不多，为什么要写多次呢？所以就搞了一个`defaultCRUD.js`来，可以直接产出一个具有CRUD属性的对象，如果可以直接用就直接用，不行就覆盖或者添加新方法。
```javascript
// src/database/defaultCRUD.js
'use strict';

import createModel from './index';
import lodash from 'lodash';

const filterInputWithSchema = (input, schema) => {
  let result = {};
  for (let key in schema) {
    if (!lodash.isEmpty(input[key])) 
      result[key] = input[key];
  }
  result.create_time = Date.now();
  result.last_edit_time = Date.now();
  return result;
};

const createDefaultCRUD = (modelName, schema) => {
  const Model = createModel(modelName, schema);
  return {
    save(data) {
      const aModel = new Model(filterInputWithSchema(data, schema));
      return aModel;
    },
    update(id, data) {
      // TODO: should generate the data according to schema
      return Model.findByIdAndUpdate(id, data);
    },
    getAll() {
      return Model.find({});
    },
    getOne(id) {
      return Model.findById(id);
    },
    deleteOne(id) {
      return Model.findByIdAndRemove(id);
    }
  };
};

export default createDefaultCRUD;
```javascript
这里需要注意的是在创建和修改的时候应该不需要接收传入的`create_time`和`last_edit_time`。再看一眼usersdb.js:
```javascript

'use strict';

import createDefaultCRUD from '../database/defaultCRUD';

const schema = {
  name: String,
  accountId: String,
  password: String,
  email: String,
  mobile: String,
  create_time: Date,
  last_edit_time: Date
};

const usersdb = createDefaultCRUD('User', schema);

export default usersdb;
```
这里就聚焦在了schema和model的名字上了。然后router里面就可以直接调用default方法：
```javascript
// src/users/router.js
router.post('/', wrap(async function(req, res, next){
  try {
    console.log('creating user', req.body);
    let result = await usersdb.save(req.body);
    res.status(200).json({id: result.id});
  } catch (err) {
    next(err);
  }
}));
```

### Get的时候不能返回password
由于牵涉到密码，所以在get的时候不能将password传出去，这里直接在router里面把拿到的user对象的password置为`undefined`：
```javascript
// src/users/router.js

router.get('/:id', wrap(async function(req, res, next){
  try {
    console.log('getting user', req.params.id);
    let result = await usersdb.getOne(req.params.id);
    console.log('user got', result, lodash.isEmpty(result));
    if (lodash.isEmpty(result))  return next(new NotFound('cannot find user with id ' + req.params.id));
    result.password = undefined;
    console.log('=====', result);
    res.status(200).json(result);
  } catch (err) {
    next(err);
  }
}));
```

### Post的时候需要对密码进行加密
关于密码加密方面的考量，可以参考这篇文章：[《Salted Password Hashing - Doing it Right》](https://crackstation.net/hashing-security.htm)，这里使用了一个库`bcrypt as promise`：
```javascript
// src/users/router.js
import bcrypt from 'bcrypt-as-promised';
router.post('/', wrap(async function(req, res, next){
  try {
    console.log('creating user', req.body);
    let bcryptedPwd = await bcrypt.hash(req.body.password, 5);
    req.body.password = bcryptedPwd;
    let result = await usersdb.save(req.body);
    console.log('result', result);
    res.status(200).json({id: result._id});
  } catch (err) {
    next(err);
  }
}));
```

package.json:
```javascript
"bcrypt-as-promised": "^1.1.0",
```


