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
```

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

这种方式当然也可以，但是不是一种最好的方式，后面会使用mongoose的方式会更好一些。

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

### Update post
在对post进行更新的时候，不需要更新`create_time`，但是需要更新`last_edit_time`。而`defaultCRUD`中的默认update方法是更新所有的数据，所以不能直接使用。那么我们需要在默认CRUD上添加一些方法：

```javascript
// src/users/usersdb.js

usersdb.updateOne = (id, data) => {
  let result = {
    "name": data.name,
    "email": data.email,
    "mobile": data.mobile,
    "last_edit_time": Date.now()
  };
  return usersdb.update(id, result);
};
```

这里我们在usersdb中添加了一个新的方法`updateOne`，这个方法只更新了部分值。其中`last_edit_time`被更新成为了当前时间。

### 修改密码
用户在修改密码的时候，需要提供之前的旧密码和新密码。在修改的时候，需要验证旧密码是否有效，如果有效的话，就更新密码：

```javascript
// src/users/router.js

router.put('/:id/password', wrap(async function(req, res, next) {
  try {
    let user = await usersdb.getOne(req.params.id);
    if (lodash.isEmpty(user)) throw new NotFound('cannot find user with ' + req.params.id);

    try { await bcrypt.compare(req.body.oldPassword, user.password); }
    catch(err) { throw new PasswordNotMatch(); }

    let newPassword = await bcrypt.hash(req.body.newPassword);
    await usersdb.updatePassword(req.params.id, newPassword);
    res.status(200).send();
  } catch (err) {
    next(err);
  }
}));
```

其中`PasswordNotMatch`是新建的Error类。依然使用了之前用于密码加密的bcrypt，只不过是用于解密了。数据库里面存储的是加密之后的密码，所以对比的时候需要先将传入的旧密码进行加密，然后对比加密之后的结果。
同样需要在usersdb中加入单独更新密码的方法：

```javascript
// src/users/usersdb.js

usersdb.updatePassword = (id, newPassword) => {
    let result = {
        "password": newPassword
    };
    return usersdb.update(id, result);
};

```

### 使用mongoose的方式不返回password
上面使用的方式是在拿到数据之后对数据进行修改，多了一次操作，同时也将密码从数据库取出来放到了内存里。在使用mongoose的时候可以通过传入参数来控制某个属性不被返回：

```javascript
// src/users/usersdb.js

usersdb.getAllWithoutPasswordField = _ => {
  return usersdb.getAll().select('-password');
};

usersdb.getOneWithoutPasswordField = userId => {
  return usersdb.getOne(userId).select('-password');
};
```

### user validation
在创建user的时候需要对传入的数据进行验证，之前在做post以及comment的时候，验证是放在具体的router里面的，这里尝试使用一个middleware来做验证。
### 加入logging
在尝试了使用winson之后，果断放弃，然后选择了log4js。
### authentication
### authorisation
