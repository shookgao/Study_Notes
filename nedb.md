## 简介
NeDB 是一个嵌入式的持久或者内存数据库，100% Javascript 编写，无二进制依赖；API 是 MongoDB 的一个子集并且它足够快。
## 安装
`npm install -S nedb`
## API
**创建/加载一个数据库**
你可以把 NeDB 创建为一个内存数据库或者持久数据库，一个数据库相当于 MongoDB 的一个集合；`new Datastore(options)`是一个构造函数，`options`是一个对象并有以下字段：
*filename*：可选，数据被持久存放的路径，如果留空，数据库会被创建在内存中。它不能以`~`结束，因为 NeDB 的临时文件要用。
*inMemoryOnly*：可选，默认`false`，就如名字所标识的那个意思。
*timestampData*：可选，默认`false`，所有文档插入和更新的时间戳，有两个字段：`createAt`和`updateAt`。用户指定的值会覆盖自动生产的值，通常用于测试。
*autoload*：可选，默认`false`，如果适用，数据库文件会被自动加载，你不需要再使用`loadDatastore`命令，所有命令在加载完成前被缓存，并在加载完成后被执行。
*onload*：可选，假如你使用自动加载，它会在`loadDatastore`后被调用，它得到一个`error`提要，当你在自动加载的时候没有指定处理器，在加载的时候发生的错误将会被抛弃。
*afterSerialization*：你可以使用这个钩子在数据被序列化后写入磁盘文件之前转换数据，比如在写入磁盘之前加密数据，这个函数接收一个字符串作为参数，并输出转换后的字符串，必须保证不出现`\n`字符，这可能会造成数据丢失。
*beforeDeserialization*：和`afterSerialization`相反，并且必须两个一起使用。
*corruptAlertThreshold*：可选，在0和1之间，默认10%，如果数据损失超过这个值，NeDB 将不能启动。
*compareStrings*：可选，函数 compareStrings(a, b) 比较字符串 a 和 b 返回-1、0、1，如果指定，它将会覆盖默认的字符串比较函数因为默认函数不能很好的适配非美国字符，特别是重音字母，原生`localCompare`在绝大多数情况下是一个更好的选择。
假如你使用一个持久数据库并且没有使用`autoload`选项，你必须手工调用`loadDatastore`，否则所有命令都将不能执行。
```
// Type 1: 内存数据库(不需要加载数据库)
var Datastore = require('nedb')
  , db = new Datastore();


// Type 2: 持久数据库手工加载
var Datastore = require('nedb')
  , db = new Datastore({ filename: 'path/to/datafile' });
db.loadDatabase(function (err) {    // Callback is optional
  // Now commands will be executed
});


// Type 3: 持久数据库自动加载
var Datastore = require('nedb')
  , db = new Datastore({ filename: 'path/to/datafile', autoload: true });
// 你现在可以马上发布命令。


// 当然如果你需要多个数据库你可以创建多个
// 集合. 在这种情况下为所有集合使用自动加载通常是个好办法.
db = {};
db.users = new Datastore('path/to/users.db');
db.robots = new Datastore('path/to/robots.db');

// 你必须加载所有数据库 (这儿我们异步处理它。)
db.users.loadDatabase();
db.robots.loadDatabase();
```
**持久化方案**
`NeDB`持久化使用附加格式，所有升级和删除的实际结果会加到数据文件的末尾。数据库自动压实。你的应用程序每次都会加载整个数据库。
压实会花费一点儿时间，在这个时间内其他操作都会被停止，因此大多数项目实际上没必要使用它。
**插入文档**
原生类型是`String`、`Number`、`Boolean`、`Date`和`null`，你也可以使用数组和子文档（对象），假如一个字段是`undefined`，它将不会被保存（这和`MongoDB`不同，它是把`undefined`转换为`null`）。
如果文档不包含`_id`字段，`NeDB`将会为你自动生成一个（一个16字符的字母数字字符串）。文档的`_id`字段一旦被设置就不能被修改。
字段名不能以`$`开头并且不能包含`.`。
```
var doc = { hello: 'world'
               , n: 5
               , today: new Date()
               , nedbIsAwesome: true
               , notthere: null
               , notToBeSaved: undefined  // 不保存
               , fruits: [ 'apple', 'orange', 'pear' ]
               , infos: { name: 'nedb' }
               };

db.insert(doc, function (err, newDoc) {   // Callback is optional
  // newDoc是最近插入的文档, 包含它的_id
  // newDoc 没有叫notToBeSaved的字段因为它的值是undefined。
});
```
你也能够插入一个文档数组，这个操作是自动的，如果数组中的一个文档插入失败，那么所有数据都将回滚。
```
db.insert([{ a: 5 }, { a: 42 }], function (err, newDocs) {
  // 两个文档被插入数据库中。
  // newDocs是这两个文档的数组, 增加了他们的_id
});

// 假如在字段`a`上有唯一约束, 操作将会失败。
db.insert([{ a: 5 }, { a: 42 }, { a: 5 }], function (err) {
  // err 是一个 'uniqueViolated' 错误
  // 数据库没有被改变
});
```
**查找文档**
使用`find`查找多个文档，使用`findOne`查找你指定的一个文档。你也能够选择文档基于字段相等或者比较操作符(`$lt`, `$lte`, `$gt`, `$gte`, `$in`, `$nin`, `$ne`)，你也能够使用逻辑操作符`$or`、`$and`、`$not`和`$where`，看下面的语法。
你有两种办法使用正则表达式：在基础查询里面替换一个字符串，或者使用`$regex`操作符。
你能够排序和分页使用游标 API。
你能够使用标准预测限制字段出现在结果中。
*基础查询*
标准查询意思是根据你指定的字段匹配查找文档。你能用正则表达式匹配字符串。为了匹配一个数组中的指定元素，你可以使用点符号在嵌入式文档、数组、子文档数组中导航。
```
// 我们的数据库中包含一下的集合
// { _id: 'id1', planet: 'Mars', system: 'solar', inhabited: false, satellites: ['Phobos', 'Deimos'] }
// { _id: 'id2', planet: 'Earth', system: 'solar', inhabited: true, humans: { genders: 2, eyes: true } }
// { _id: 'id3', planet: 'Jupiter', system: 'solar', inhabited: false }
// { _id: 'id4', planet: 'Omicron Persei 8', system: 'futurama', inhabited: true, humans: { genders: 7 } }
// { _id: 'id5', completeData: { planets: [ { name: 'Earth', number: 3 }, { name: 'Mars', number: 2 }, { name: 'Pluton', number: 9 } ] } }

// 在太阳系中找出所有行星
db.find({ system: 'solar' }, function (err, docs) {
  // docs 是一个包含 Mars, Earth, Jupiter 的数组
  // 假如没有文档被发现, docs 等于 []
});

// 使用正则表达式找出名称中包含 ar 的行星
db.find({ planet: /ar/ }, function (err, docs) {
  // docs 包含 Mars 和 Earth
});

// 找出太阳系中有人居住的行星
db.find({ system: 'solar', inhabited: true }, function (err, docs) {
  // docs 是一个仅仅包含文档 Earth 的数组
});

// 使用点符号在子文档中匹配字段
db.find({ "humans.genders": 2 }, function (err, docs) {
  // docs 包含 Earth
});

// 使用点符号在子文档数组中导航
db.find({ "completeData.planets.name": "Mars" }, function (err, docs) {
  // docs 包含文档 5
});

db.find({ "completeData.planets.name": "Jupiter" }, function (err, docs) {
  // docs 是空
});

db.find({ "completeData.planets.0.name": "Earth" }, function (err, docs) {
  // docs 包含文档 5
  // 假如我们把字段值换成 "Mars" docs 将是空的因为我们匹配了一个指定的数组元素。
});


// 你也可以深度比较对象. 不要使用点符号!
db.find({ humans: { genders: 2 } }, function (err, docs) {
  // docs 是空的, 因为 { genders: 2 } 不等于 { genders: 2, eyes: true }
});

// 在这个集合中查找所有文档
db.find({}, function (err, docs) {
});

// 当你想找一个文档的时候使用 findOne，使用规则一样
db.findOne({ _id: 'id1' }, function (err, doc) {
  // doc 是文档 Mars
  // 假如没有文档被发现, doc 是 null
});
```
*操作符 (`$lt`, `$lte`, `$gt`, `$gte`, `$in`, `$nin`, `$ne`, `$exists`, `$regex`)*
这个语法是`{ field: { $op: value } }`，`$op`是任意比较操作符。

- `$lt`,`$lte`：小于，小于或等于
- `$gt`,`$gte`：大于，大于或等于
- `$in`：`.value`的成员，必须是一个数组值
- `$ne`,`nin`：不等于，不是一个成员
- `$exists`：检查字段是否存在，值是 true 或者 false
- `$regex`：检查一个字符串是否被正则表达式匹配。和 `MongoDB`相反，`NeDB`的`$regex`不支持`$options`，因为它并没有什么卵用。
```
// $lt, $lte, $gt and $gte 工作在数字和字符串上
db.find({ "humans.genders": { $gt: 5 } }, function (err, docs) {
  // docs 包含 Omicron Persei 8, 它的 humans.genders 是7，大于5.
});

// 当使用在字符串上的时候，使用字典排序
db.find({ planet: { $gt: 'Mercury' }}, function (err, docs) {
  // docs contains Omicron Persei 8
})

// 使用 $in. $nin 的方法是一样的
db.find({ planet: { $in: ['Earth', 'Jupiter'] }}, function (err, docs) {
  // docs 包含 Earth and Jupiter
});

// 使用 $exists
db.find({ satellites: { $exists: true } }, function (err, docs) {
  // docs 只包含 Mars
});

// 有多个操作符的时候，正则表达式要使用操作符 $regex 
db.find({ planet: { $regex: /ar/, $nin: ['Jupiter', 'Earth'] } }, function (err, docs) {
  // docs 仅仅包含 Mars 因为 Earth 被 $nin 匹配了
});
```
*数组字段*
当一个文档的值是一个数组时，`NeDB`首先会看查询的值是不是数组，是的话就进行一个精确匹配，然后看是不是有数组比较函数，如果也没有，将会进行模糊匹配。
- `$size`：匹配数组的大小
- `$elemMatch`：至少有一个元素匹配
```
// 精确匹配
db.find({ satellites: ['Phobos', 'Deimos'] }, function (err, docs) {
  // docs 包含 Mars
})
db.find({ satellites: ['Deimos', 'Phobos'] }, function (err, docs) {
  // docs 是空的，不同顺序也不行
})

// 使用指定的数组匹配函数
// $elemMatch 操作符为一个文档提供匹配, 假如数组中的元素满足 `$elemMatch` 操作符的所有约束
db.find({ completeData: { planets: { $elemMatch: { name: 'Earth', number: 3 } } } }, function (err, docs) {
  // docs 包含文档 id 5 (completeData)
});

db.find({ completeData: { planets: { $elemMatch: { name: 'Earth', number: 5 } } } }, function (err, docs) {
  // docs 是空的
});

// 你能够在 #elemMatch 查询里面使用任何的文档操作符
db.find({ completeData: { planets: { $elemMatch: { name: 'Earth', number: { $gt: 2 } } } } }, function (err, docs) {
  // docs 包含文档 id 5 (completeData)
});

// 注意：你不能在 $size 操作符里面使用比较函数, e.g. { $size: { $lt: 5 } } 将会抛出一个错误
db.find({ satellites: { $size: 2 } }, function (err, docs) {
  // docs 包含 Mars
});

db.find({ satellites: { $size: 1 } }, function (err, docs) {
  // docs 是空的
});

// 假如文档的字段是一个数组, 匹配一个意味着匹配数组中的任意一个。
db.find({ satellites: 'Phobos' }, function (err, docs) {
  // docs 包含 Mars. 使用 { satellites: 'Deimos' } 匹配是一样的结果
});

// 这种查询的方式也可以使用比较函数
db.find({ satellites: { $lt: 'Amos' } }, function (err, docs) {
  // docs 是空的
});

// $in and $nin 操作符也可以工作
db.find({ satellites: { $in: ['Moon', 'Deimos'] } }, function (err, docs) {
  // docs 包含 Mars (Earth 文档是不完整的!)
});
```

*逻辑操作符 `$or`, `$and`, `$not`, `$where`*
你能够使用逻辑操作符组合查询：
- `$or`,`$and`：语法是`{ $op: [query1, query2, ...] }`
- `$not`：语法是`{ $not: query }`
- `$where`：语法是`{ $where: function () { /* object is "this", return a boolean */ } }`
```
db.find({ $or: [{ planet: 'Earth' }, { planet: 'Mars' }] }, function (err, docs) {
  // docs 包含 Earth 和 Mars
});

db.find({ $not: { planet: 'Earth' } }, function (err, docs) {
  // docs 包含 Mars, Jupiter, Omicron Persei 8
});

db.find({ $where: function () { return Object.keys(this) > 6; } }, function (err, docs) {
  // docs 字段大于6个
});

// 你能够混合普通查询、比较函数和逻辑操作符
db.find({ $or: [{ planet: 'Earth' }, { planet: 'Mars' }], inhabited: true }, function (err, docs) {
  // docs contains Earth
});
```
*排序和分页*
假如你不给 find,findOne,count 指定回调函数，一个游标对象将被返回，你能够用 sort,skip,limit 修改游标后使用`exec(callback)`执行它。
```
// 我们的数据库包含4个文档
// doc1 = { _id: 'id1', planet: 'Mars', system: 'solar', inhabited: false, satellites: ['Phobos', 'Deimos'] }
// doc2 = { _id: 'id2', planet: 'Earth', system: 'solar', inhabited: true, humans: { genders: 2, eyes: true } }
// doc3 = { _id: 'id3', planet: 'Jupiter', system: 'solar', inhabited: false }
// doc4 = { _id: 'id4', planet: 'Omicron Persei 8', system: 'futurama', inhabited: true, humans: { genders: 7 } }

// 没有查询条件意味着所有结果被返回 (在游标修改前)
db.find({}).sort({ planet: 1 }).skip(1).limit(2).exec(function (err, docs) {
  // docs 是 [doc3, doc1]
});

// 你可以像下面这样反向排序
db.find({ system: 'solar' }).sort({ planet: -1 }).exec(function (err, docs) {
  // docs is [doc1, doc3, doc2]
});

// 你可以排序一个字段，然后另一个字段，像下面那样操作:
db.find({}).sort({ firstField: 1, secondField: -1 }) ...   // You understand how this works!
```
*预测*
你能够给 find, findOne 第二个可选的参数，`projections`，这个语法和`MongoDB`一样：`{ a: 1, b: 1 }`表明要返回这两个字段，`{ a: 0, b: 0 }`表明不要返回这两个字段，这两个方式不能同时使用，除了`_id`这个字段总是会返回，你可以省略它。
```
// 和上面的数据库一样

// 仅仅要给出的字段
db.find({ planet: 'Mars' }, { planet: 1, system: 1 }, function (err, docs) {
  // docs is [{ planet: 'Mars', system: 'solar', _id: 'id1' }]
});

// 保持给出的字段并省略_id
db.find({ planet: 'Mars' }, { planet: 1, system: 1, _id: 0 }, function (err, docs) {
  // docs is [{ planet: 'Mars', system: 'solar' }]
});

// 省略给出的字段
db.find({ planet: 'Mars' }, { planet: 0, system: 0, _id: 0 }, function (err, docs) {
  // docs is [{ inhabited: false, satellites: ['Phobos', 'Deimos'] }]
});

// 失败: 在同一次使用了两个方式
db.find({ planet: 'Mars' }, { planet: 0, system: 1 }, function (err, docs) {
  // err is the error message, docs is undefined
});

// 你也可以在游标中使用这个方式，但是和 MongoDB 不兼容
db.find({ planet: 'Mars' }).projection({ planet: 1, system: 1 }).exec(function (err, docs) {
  // docs is [{ planet: 'Mars', system: 'solar', _id: 'id1' }]
});

// 在嵌入文档中使用
db.findOne({ planet: 'Earth' }).projection({ planet: 1, 'humans.genders': 1 }).exec(function (err, doc) {
  // doc is { planet: 'Earth', _id: 'id2', humans: { genders: 2 } }
});
```
*文档计数*
你能够使用 count 来计数，它的语法和 find 一样，例如：
```
// 计数太阳系中所有行星
db.count({ system: 'solar' }, function (err, count) {
  // count 等于 3
});

// Count 数据库中所有文档
db.count({}, function (err, count) {
  // count 等于 4
});
```
**更新文档**
`db.update(query, update, options, callback)`将会升级所有匹配`query`的文档的`update`:

- `query`的方法和`find`方法一样。
- `update`指定要更新什么内容，这是一个新文档或者一组字段修改。
    - 新文档会替换旧文档
    - 如果字段不存在会新增，可用字段修改使用`$set`，`$unset`会删除一个字段，`$inc`增加一个字段的值，还有`$min`和`$max`不知道有什么用；为了工作在数组上，你还可用`$push`, `$pop`, `$addToSet`, `$pull`, `$each` and `$slice`。
- `options`是有两个可能参数的对象
    - `multi`：默认是`false`，同时修改多个文档
    - `upsert`：默认是`false`
    - `returnUpdatedDocs`：默认是`false`，和`MongoDB`不兼容
- `callback`：可选，(err, numAffected, affectedDocuments, upsert)
    - 在upsert时，`affectedDocuments`包含插入的文档，upsert标志是true
    - 在标准更新中并且`returnUpdatedDocs`设置为`false`时，`affectedDocuments`不会被设置。
    - 在标准更新中并且`returnUpdatedDocs`设置为true并且`multi`为`false`时，`affectedDocuments`是更新文档
    - 在标准更新中并且`returnUpdatedDocs`设置为true并且`multi`为`true`时，`affectedDocuments`是更新文档数组。
注意：你不能改变一个文档的_id
```
// 我们的数据库是这样
// { _id: 'id1', planet: 'Mars', system: 'solar', inhabited: false }
// { _id: 'id2', planet: 'Earth', system: 'solar', inhabited: true }
// { _id: 'id3', planet: 'Jupiter', system: 'solar', inhabited: false }
// { _id: 'id4', planet: 'Omicron Persia 8', system: 'futurama', inhabited: true }

// 替换一个文档
db.update({ planet: 'Jupiter' }, { planet: 'Pluton'}, {}, function (err, numReplaced) {
  // numReplaced = 1
  // 文档 #3 已经被替换成 { _id: 'id3', planet: 'Pluton' }
  // 注意这个`_id`是保持没变的
  // (这个`system`和`inhabited`字段不在这儿了)
});

// 设置一个已存在字段的值
db.update({ system: 'solar' }, { $set: { system: 'solar system' } }, { multi: true }, function (err, numReplaced) {
  // numReplaced = 3
  // 字段 'system' 在 Mars, Earth, Jupiter 里面现在是 'solar system'
});

// 使用点符号在一个子文档里面设置一个不存在的字段的值
db.update({ planet: 'Mars' }, { $set: { "data.satellites": 2, "data.red": true } }, {}, function () {
  // Mars 文档现在是 { _id: 'id1', system: 'solar', inhabited: false
  //                      , data: { satellites: 2, red: true }
  //                      }
  // 子文档中设置字段，你必须使用点符号
  // 使用对象符号仅仅能够替换顶层字段
  db.update({ planet: 'Mars' }, { $set: { data: { satellites: 3 } } }, {}, function () {
    // Mars 现在是 { _id: 'id1', system: 'solar', inhabited: false
    //                      , data: { satellites: 3 }
    //                      }
    // 你失去了 "data.red" 字段，这可能不是你想要的
  });
});

// 删除一个字段
db.update({ planet: 'Mars' }, { $unset: { planet: true } }, {}, function () {
  // 现在 Mars 文档不包含 planet 字段
  // 你可以使用点符号删除一个嵌入的字段
});

// 更新插入文档
db.update({ planet: 'Pluton' }, { planet: 'Pluton', inhabited: false }, { upsert: true }, function (err, numReplaced, upsert) {
  // numReplaced = 1, upsert = { _id: 'id5', planet: 'Pluton', inhabited: false }
  // 一个新的文档 { _id: 'id5', planet: 'Pluton', inhabited: false } 已经被加到集合中。
});

// 假如你插入一个修改, 这个插入的文档是查询修改
// 这比听起来更简单 :)
db.update({ planet: 'Pluton' }, { $inc: { distance: 38 } }, { upsert: true }, function () {
  // 一个新文档 { _id: 'id5', planet: 'Pluton', distance: 38 } 被加到这个集合  
});

// 假如我们在集合中插入一个 { _id: 'id6', fruits: ['apple', 'orange', 'pear'] } 文档,
// 让我们看一下我们怎么修改数组字段

// $push 在数组的结尾插入新的值
db.update({ _id: 'id6' }, { $push: { fruits: 'banana' } }, {}, function () {
  // 现在这个fruits数组是 ['apple', 'orange', 'pear', 'banana']
});

// $pop 移除一个元素，从结尾移除用1，从开始移除使用-1
db.update({ _id: 'id6' }, { $pop: { fruits: 1 } }, {}, function () {
  // 现在 fruits 数组是 ['apple', 'orange']
  // W使用 { $pop: { fruits: -1 } }, 结果会变成 ['orange', 'pear']
});

// $addToSet adds an element to an array only if it isn't already in it
// Equality is deep-checked (i.e. $addToSet will not insert an object in an array already containing the same object)
// Note that it doesn't check whether the array contained duplicates before or not
db.update({ _id: 'id6' }, { $addToSet: { fruits: 'apple' } }, {}, function () {
  // The fruits array didn't change
  // If we had used a fruit not in the array, e.g. 'banana', it would have been added to the array
});

// $pull removes all values matching a value or even any NeDB query from the array
db.update({ _id: 'id6' }, { $pull: { fruits: 'apple' } }, {}, function () {
  // Now the fruits array is ['orange', 'pear']
});
db.update({ _id: 'id6' }, { $pull: { fruits: $in: ['apple', 'pear'] } }, {}, function () {
  // Now the fruits array is ['orange']
});

// $each can be used to $push or $addToSet multiple values at once
// This example works the same way with $addToSet
db.update({ _id: 'id6' }, { $push: { fruits: { $each: ['banana', 'orange'] } } }, {}, function () {
  // Now the fruits array is ['apple', 'orange', 'pear', 'banana', 'orange']
});

// $slice can be used in cunjunction with $push and $each to limit the size of the resulting array.
// A value of 0 will update the array to an empty array. A positive value n will keep only the n first elements
// A negative value -n will keep only the last n elements.
// If $slice is specified but not $each, $each is set to []
db.update({ _id: 'id6' }, { $push: { fruits: { $each: ['banana'], $slice: 2 } } }, {}, function () {
  // Now the fruits array is ['apple', 'orange']
});

// $min/$max to update only if provided value is less/greater than current value
// Let's say the database contains this document
// doc = { _id: 'id', name: 'Name', value: 5 }
db.update({ _id: 'id1' }, { $min: { value: 2 } }, {}, function () {
  // The document will be updated to { _id: 'id', name: 'Name', value: 2 }
});

db.update({ _id: 'id1' }, { $min: { value: 8 } }, {}, function () {
  // The document will not be modified
});
```