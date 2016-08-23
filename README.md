# Normalizr-translate-chinese
[官方英文文档](https://github.com/paularmstrong/normalizr)

翻译的不太好，可以对比着看看
# normalizr [![build status](https://img.shields.io/travis/paularmstrong/normalizr/master.svg?style=flat-square)](https://travis-ci.org/paularmstrong/normalizr) [![npm version](https://img.shields.io/npm/v/normalizr.svg?style=flat-square)](https://www.npmjs.com/package/normalizr) [![npm downloads](https://img.shields.io/npm/dm/normalizr.svg?style=flat-square)](https://www.npmjs.com/package/normalizr)

在[Flux](https://facebook.github.io/flux)和[Redux](http://rackt.github.io/redux) 应用中根据一种模式来扁平化深层嵌套的api返回的json数据

## 安装

```
npm install --save normalizr
```

## 应用例子

### Flux

See **[flux-react-router-example](https://github.com/gaearon/flux-react-router-example)**.

### Redux

See **[redux/examples/real-world](https://github.com/rackt/redux/tree/master/examples/real-world)**.

## 问题
* 你有一个返回了深层嵌套的对象的API请求
* 你想要你的应用针对[Flux](https://github.com/facebook/flux) 或者 [Redux](http://rackt.github.io/redux);
* 你发现Store(或者Reducers)从深层嵌套的数据中获取和操作数据会非常麻烦
 
Normalizr采用JSON格式和一种schema，并且用嵌套实体的id来替换嵌套实体，整合所有的实体到字典对象中

For example,

```javascript
[{
  id: 1,
  title: 'Some Article',
  author: {
    id: 1,
    name: 'Dan'
  }
}, {
  id: 2,
  title: 'Other Article',
  author: {
    id: 1,
    name: 'Dan'
  }
}]
```

可以被范式化成(normalized to)

```javascript
{
  result: [1, 2],
  entities: {
    articles: {
      1: {
        id: 1,
        title: 'Some Article',
        author: 1
      },
      2: {
        id: 2,
        title: 'Other Article',
        author: 1
      }
    },
    users: {
      1: {
        id: 1,
        name: 'Dan'
      }
    }
  }
}
```


注意它的扁平化结构(所有的嵌套都没了)

## 特性
* 实体可以嵌套在其他实体、对象、数组中
* 可以通过结合实体的schema来表示任何类型API返回的数据
* 相同id的实体会自动合并 (with a warning if they differ);
* 允许使用自定义的id属性作为合并的依据 (e.g. slug).

## 用法

```javascript
import { normalize, Schema, arrayOf } from 'normalizr';
```

首先，为实体定义一个模式(schema)

```javascript
const article = new Schema('articles');
const user = new Schema('users');
```

然后我们定义一个嵌套规则

```javascript
article.define({
  author: user,
  contributors: arrayOf(user)
});
```

现在我们可以在处理API请求返回数据的方法中使用这个schema:

```javascript
const ServerActionCreators = {

  // 有两个带有不同响应对象的schemas的XHR endpoints----------------------------
  // 我们可以用前面定义的schema对象来表示他们:

  receiveOneArticle(response) {

    // 在这里，这个返回的数据是一个包含了一篇文章的对象
    // 把文章的schema作为normalize方法的第二个参数，让他能正确的遍历响应对象并且把所有的实体都整合到一起

    // BEFORE:
    // {
    //   id: 1,
    //   title: 'Some Article',
    //   author: {
    //     id: 7,
    //     name: 'Dan'
    //   },
    //   contributors: [{
    //     id: 10,
    //     name: 'Abe'
    //   }, {
    //     id: 15,
    //     name: 'Fred'
    //   }]
    // }
    //
    // AFTER:
    // {
    //   result: 1,                    // <--- 注意对象是由ID来引用的
    //   entities: {
    //     articles: {
    //       1: {
    //         author: 7,              // <--- 同样适用于在其它实体中的引用
    //         contributors: [10, 15]  
    //         ...}
    //     },
    //     users: {
    //       7: { ... },
    //       10: { ... },
    //       15: { ... }
    //     }
    //   }
    // }

    response = normalize(response, article);

    AppDispatcher.handleServerAction({
      type: ActionTypes.RECEIVE_ONE_ARTICLE,
      response
    });
  },

  receiveAllArticles(response) {

    // 这里，返回的数据是一个对象，他的key为'articles'并且该key指向了一个包含文章对象的数组
    // 把 { articles: arrayOf(article) }作为normalize方法的第二个参数，让他能正确的遍历响应对象并且把所有的实体都整合到一起

    // BEFORE:
    // {
    //   articles: [{
    //     id: 1,
    //     title: 'Some Article',
    //     author: {
    //       id: 7,
    //       name: 'Dan'
    //     },
    //     ...
    //   },
    //   ...
    //   ]
    // }
    //
    // AFTER:
    // {
    //   result: {
    //    articles: [1, 2, ...]     // <--- 注意对象数组转换成ID数组的方式
    //   },
    //   entities: {
    //     articles: {
    //       1: { author: 7, ... }, // <--- 同样适用于在其它实体中的引用
    //       2: { ... },
    //       ...
    //     },
    //     users: {
    //       7: { ... },
    //       ..
    //     }
    //   }
    // }

    response = normalize(response, {
      articles: arrayOf(article)
    });

    AppDispatcher.handleServerAction({
      type: ActionTypes.RECEIVE_ALL_ARTICLES,
      response
    });
  }
}
```

Finally, different Stores can tune in to listen to all API responses and grab entity lists from `action.response.entities`:

```javascript
AppDispatcher.register((payload) => {
  const { action } = payload;

  if (action.response && action.response.entities && action.response.entities.users) {
    mergeUsers(action.response.entities.users);
    UserStore.emitChange();
    break;
  }
});
```

## API Reference

### `new Schema(key, [options])`

Schema 允许你定义一个通过API返回的实体的类型
这应该与你服务器上代码的模型相对应

`key`参数允许你为这类实体指定一个名字(译者的理解其实就是数据库表名)

```javascript
const article = new Schema('articles');

// 你可以使用自定义的id属性
const article = new Schema('articles', { idAttribute: 'slug' });

// 或者指定一个函数来生成
function generateSlug(entity) { /* ... */ }
const article = new Schema('articles', { idAttribute: generateSlug });

// You can also specify meta properties to be used for customizing the output in assignEntity (见下)
const article = new Schema('articles', { idAttribute: 'slug', meta: { removeProps: ['publisher'] }});

// You can specify custom `assignEntity` function to be run after the `assignEntity` function passed to `normalize`
const article = new Schema('articles', { assignEntity: function (output, key, value, input) {
  if (key === 'id_str') {
    output.id = value;
    if ('id_str' in output) {
      delete output.id_str;
    }
  } else {
    output[key] = value;
  }
}})

// 你可以为实体指定默认的值
const article = new Schema('articles', { defaults: { likes: 0 } });
```

### `Schema.prototype.define(nestedSchema)`

允许你指定不同实体之间的关系

```javascript
const article = new Schema('articles');
const user = new Schema('users');

article.define({
  author: user
});
```

### `Schema.prototype.getKey()`

返回这个schema的key

```javascript
const article = new Schema('articles');

article.getKey();
// articles
```

### `Schema.prototype.getIdAttribute()`

返回这个schema的id属性

```javascript
const article = new Schema('articles');
const slugArticle = new Schema('articles', { idAttribute: 'slug' });

article.getIdAttribute();
// id
slugArticle.getIdAttribute();
// slug
```

### `Schema.prototype.getDefaults()`

返回这个schema的默认值

```javascript
const article = new Schema('articles', { defaults: { likes: 0 } });

article.getDefaults();
// { likes: 0 }
```

### `arrayOf(schema, [options])`

描述了一个schema的数组作为参数被传递

```javascript
const article = new Schema('articles');
const user = new Schema('users');

article.define({
  author: user,
  contributors: arrayOf(user)
});
```

If the array contains entities with different schemas, you can use the `schemaAttribute` option to specify which schema to use for each entity:
如果一个数组包含了具有不同schema的实体，你可以使用`schemaAttribute`选项来为每一个实体指定schema

```javascript
const article = new Schema('articles');
const image = new Schema('images');
const video = new Schema('videos');
const asset = {
  images: image,
  videos: video
};

// You can specify the name of the attribute that determines the schema
article.define({
  assets: arrayOf(asset, { schemaAttribute: 'type' })
});

// Or you can specify a function to infer it
function inferSchema(entity) { /* ... */ }
article.define({
  assets: arrayOf(asset, { schemaAttribute: inferSchema })
});
```

### `valuesOf(schema, [options])`

Describes a map whose values follow the schema passed as argument.

```javascript
const article = new Schema('articles');
const user = new Schema('users');

article.define({
  collaboratorsByRole: valuesOf(user)
});
```

If the map contains entities with different schemas, you can use the `schemaAttribute` option to specify which schema to use for each entity:

```javascript
const article = new Schema('articles');
const user = new Schema('users');
const group = new Schema('groups');
const collaborator = {
  users: user,
  groups: group
};

// You can specify the name of the attribute that determines the schema
article.define({
  collaboratorsByRole: valuesOf(collaborator, { schemaAttribute: 'type' })
});

// Or you can specify a function to infer it
function inferSchema(entity) { /* ... */ }
article.define({
  collaboratorsByRole: valuesOf(collaborator, { schemaAttribute: inferSchema })
});
```

### `unionOf(schemaMap, [options])`

Describe a schema which is a union of multiple schemas.  This is useful if you need the polymorphic behavior provided by `arrayOf` or `valuesOf` but for non-collection fields.

Use the required `schemaAttribute` option to specify which schema to use for each entity.

```javascript
const group = new Schema('groups');
const user = new Schema('users');

// a member can be either a user or a group
const member = {
  users: user,
  groups: group
};

// You can specify the name of the attribute that determines the schema
group.define({
  owner: unionOf(member, { schemaAttribute: 'type' })
});

// Or you can specify a function to infer it
function inferSchema(entity) { /* ... */ }
group.define({
  creator: unionOf(member, { schemaAttribute: inferSchema })
});
```

A `unionOf` schema can also be combined with `arrayOf` and `valuesOf` with the same behavior as each supplied with the `schemaAttribute` option.

```javascript
const group = new Schema('groups');
const user = new Schema('users');

const member = unionOf({
  users: user,
  groups: group
}, { schemaAttribute: 'type' });

group.define({
  owner: member,
  members: arrayOf(member),
  relationships: valuesOf(member)
});
```

### `normalize(obj, schema, [options])`

Normalizes object according to schema.  
Passed `schema` should be a nested object reflecting the structure of API response.

You may optionally specify any of the following options:

* `assignEntity` (function): This is useful if your backend emits additional fields, such as separate ID fields, you'd like to delete in the normalized entity. See [the tests](https://github.com/gaearon/normalizr/blob/a0931d7c953b24f8f680b537b5f23a20e8483be1/test/index.js#L89-L200) and the [discussion](https://github.com/gaearon/normalizr/issues/10) for a usage example.

* `mergeIntoEntity` (function): You can use this to resolve conflicts when merging entities with the same key. See [the test](https://github.com/gaearon/normalizr/blob/47ed0ecd973da6fa7c8b2de461e35b293ae52047/test/index.js#L132-L197) and the [discussion](https://github.com/gaearon/normalizr/issues/34) for a usage example.

```javascript
const article = new Schema('articles');
const user = new Schema('users');

article.define({
  author: user,
  contributors: arrayOf(user),
  meta: {
    likes: arrayOf({
      user: user
    })
  }
});

// ...

// Normalize one article object
const json = { id: 1, author: ... };
const normalized = normalize(json, article);

// Normalize an array of article objects
const arr = [{ id: 1, author: ... }, ...]
const normalized = normalize(arr, arrayOf(article));

// Normalize an array of article objects, referenced by an object key:
const wrappedArr = { articles: [{ id: 1, author: ... }, ...] }
const normalized = normalize(wrappedArr, {
  articles: arrayOf(article)
});
```

## 通过例子来解释

假如你有返回以下schema的API请求`/articles`:

```
articles: article*

article: {
  author: user,
  likers: user*
  primary_collection: collection?
  collections: collection*
}

collection: {
  curator: user
}
```

如果不做范式化，你的store需要清楚的知道返回数据的结构
比如， `UserStore`在获取到请求结果的时候会包含很多样板代码来获取新用户

```javascript
// 如果不做范式化, 你需要对每一个store都做这些
AppDispatcher.register((payload) => {
  const { action } = payload;

  switch (action.type) {
  case ActionTypes.RECEIVE_USERS:
    mergeUsers(action.rawUsers);
    break;

  case ActionTypes.RECEIVE_ARTICLES:
    action.rawArticles.forEach(rawArticle => {
      mergeUsers([rawArticle.user]);
      mergeUsers(rawArticle.likers);

      mergeUsers([rawArticle.primaryCollection.curator]);
      rawArticle.collections.forEach(rawCollection => {
        mergeUsers(rawCollection.curator);
      });
    });

    UserStore.emitChange();
    break;
  }
});
```

Normalizr解决了这个问题，通过把嵌套的实体用id替代把请求返回的数据变成了一个扁平化的结构的对象

```javascript
{
  result: [12, 10, 3, ...],
  entities: {
    articles: {
      12: {
        authorId: 3,
        likers: [2, 1, 4],
        primaryCollection: 12,
        collections: [12, 11]
      },
      ...
    },
    users: {
      3: {
        name: 'Dan'
      },
      2: ...,
      4: ....
    },
    collections: {
      12: {
        curator: 2,
        name: 'Stuff'
      },
      ...
    }
  }
}
```

`UserStore`的代码可以像这样被重写

```javascript
// 做了范式化后，所有的用户都在action.response.entities.users中

AppDispatcher.register((payload) => {
  const { action } = payload;

  if (action.response && action.response.entities && action.response.entities.users) {
    mergeUsers(action.response.entities.users);
    UserStore.emitChange();
    break;
  }
});
```

## 依赖

* 一些方法依赖 `lodash`, 比如 `isObject`, `isEqual` 和 `mapValues`

## Browser Support

Modern browsers with ES5 environments are supported.  
The minimal supported IE version is IE 9.

## Running Tests

```
git clone https://github.com/gaearon/normalizr.git
cd normalizr
npm install
npm test # run tests once
npm run test:watch # run test watcher
```

## Credits

Normalizr was originally created by [Dan Abramov](http://github.com/gaearon) and inspired by a conversation with [Jing Chen](https://twitter.com/jingc).  
It has since received contributions from different [community members](https://github.com/gaearon/normalizr/graphs/contributors).

