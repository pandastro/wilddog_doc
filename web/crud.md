/*
Title: 读写数据
Sort: 1
Tmpl : page
*/

## 写入数据
写入数据的工作可以通过三个函数来实现：`set()`, `update()`, `push()`。它们的作用分别是：
* `set()`：用于替换当前节点下的数据。
* `update()`：修改子节点的数据。
* `push()`：在当前节点下新增数据，但数据的key随机生成 


#### 用 set() 写入数据

set() 是最基本的写数据操作，它设置当前指定数据节点的值，如果当前节点已经存在，旧值将被覆盖。为了理解 set() 的工作原理，我们创建一个简单的博客应用，这个博客的数据储存在这里：

```js
var ref = new Wilddog("https://docs-examples.wilddogio.com/web/saving-data/wildblog");
```
我们用用户名来唯一标识一个用户，并存储他们的全名和生日。我们已经创建了一个引用，接下来用 set() 储存数据，set() 可以传入 `string`, `number`, `boolean`, `object` 类型:
```js
var usersRef = ref.child("users");
usersRef.set({
  alanisawesome: {
    date_of_birth: "June 23, 1912",
    full_name: "Alan Turing"
  },

  gracehop: {
    date_of_birth: "December 9, 1906",
    full_name: "Grace Hopper"
  }
});

```

数据被保存到了相应的位置上。实现同样的效果，也可以这样做:

```js
usersRef.child("alanisawesome").set({
  date_of_birth: "June 23, 1912",
  full_name: "Alan Turing"
});

usersRef.child("gracehop").set({
  date_of_birth: "December 9, 1906",
  full_name: "Grace Hopper"
});

```

这两种方式的区别是，如果 /users 路径下原本已有数据存在，第一种方式会把数据全部覆盖掉，而第二种方式只会覆盖 `alanisawesome` 和`gracehop` 两个子节点，不会删除其它已有的子节点。



#### 使用 update() 更新数据

如果要同时更新多个子节点，而不覆盖其它的子节点，可以使用 update() 方法:

```js
var hopperRef = usersRef.child("gracehop");

hopperRef.update({
  "nickname": "Amazing Grace"
});

```

这样会更新 Grace 的 nickname 字段。如果我们用 set() 而不是 update()，那么 date_of_birth 和 full_name 都会被删除。


#### 使用 push() 保存一个列表
当多个用户同时试图在一个节点下新增一个子节点的时候，这时，数据就会被重写，覆盖。
为了解决这个问题，Wilddog push()采用了生成唯一ID 作为key的方式。通过这种方式，多个用户同时在一个节点下面push 数据，他们的key一定是不同的。这个key是通过一个基于时间戳和随机的算法生成的。wilddog采用了足够多的位数保证唯一性。

用户可以用push向博客app中写新内容：


```js
  var postsRef = ref.child("posts");

  postsRef.push({
    author: "gracehop",
    title: "Announcing COBOL, a New Programming Language"
  });

  postsRef.push({
    author: "alanisawesome",
    title: "The Turing Machine"
  });

```

产生的数据有一个唯一ID:
```json
{

  "posts": {
    "-JRHTHaIs-jNPLXO": {
      "author": "gracehop",
      "title": "Announcing COBOL, a New Programming Language"
    },

    "-JRHTHaKuITFIhnj": {
      "author": "alanisawesome",
      "title": "The Turing Machine"
    }
  }
}
```

**获取唯一ID**
调用push会返回一个引用，这个引用指向新增数据所在的节点。你可以通过调用 `key()` 来获取这个唯一ID

```js
 // 通过push()来获得一个新的数据库地址
var newPostRef = postsRef.push({
	author: "gracehop",
	title: "Announcing COBOL, a New Programming Language"
});

// 获取push()生成的唯一ID
var postID = newPostRef.key();

```

<hr>

## 事件类型

Wilddog提供了四种数据事件：`value`，`child_added`，`child_changed`，`child_removed`。

**value**
`value` 事件用来读取当前节点的静态数据快照， `value` 事件在初次获取到数据时被触发一次，此后每当数据发生变化都会被触发。回调方法被执行时候，当前节点下所有数据的静态快照会被作为参数传入。


**child_added**
与`value`事件返回指定数据节点下的所有数据不同，`child_added`事件会为每一个现存的子节点数据触发一次，此后每当有子节点数据被增加时被触发一次。回调方法传入的也是DataSnapshot对象，包含的是新增子节点的数据。

如果我们只想获取新增的博客数据，可以使用 `child_added`事件:

```js
// 获得一个数据库连接实例
var ref = new Wilddog("https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts");

// 获得新增加的数据
ref.on("child_added", function(snapshot,error) {
	if (error == null) {
		var newPost = snapshot.val();
		console.log("Author: " + newPost.author);
		console.log("Title: " + newPost.title);
	}
});

```

**child_changed**
当子节点数据发生了改变时，`child_changed`事件会被触发，事件回调方法中传入的参数是子节点改变后的数据快照。
我们可以使用`child_changed`事件来读取被修改的博客数据：

```js
// 获得一个数据库连接实例
var ref = new Wilddog("https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts");
// 获得发生改变的数据
ref.on("child_changed", function(snapshot) {
  var changedPost = snapshot.val();
  console.log("The updated post title is " + changedPost.title);
});
```

**child_removed**
当直接子节点被删除时，`child_removed` 事件被触发。事件回调方法中传入的参数是被删除的直接子节点的数据快照。

在博客的例子中，我们可以使用`child_removed`事件，在有博客数据被删除时，在控制台中打印出来：

```js
// 获得一个数据库连接实例
var ref = new Wilddog("https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts");
// 获取被删除的数据
ref.on("child_removed", function(snapshot) {
  var deletedPost = snapshot.val();
  console.log("The blog post titled '" + deletedPost.title + "' has been deleted");
});

```

<hr>


## 读取数据
Wilddog查询数据的方式是绑定一个异步监听的回调方法，每当数据被第一次查询或者数据发生变化，这个回调方法都会被执行。

重新看博客app的例子，我们可以这样读取博客数据:

```js
// 获得一个数据库连接实例
var ref = new Wilddog("https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts");

// 监听数据
ref.on("value", function(snapshot) {
	console.log(snapshot.val());
}, function (errorObject) {
	console.log("The read failed: " + errorObject.code);
});

```

运行这段代码，我们会看到控制台中打印出一个对象，包含所有的博客数据。每次当有新的博客数据被写入时，这个回调方法都会被触发执行。

回调方法接收一个Snapshot类型作为参数。一个Snapshot对象代表的是在指定的数据路径下，某一个时间点上数据的一个快照。调用它的`val()`方法，可以得到一个代表当前节点数据的javascript对象。如果路径节点下没有数据，那么这个Snapshot将为null。

这个例子中我们用 `'value'` 作为事件类型，用于读取当前路径节点下的所有数据。我们还可以使用另外三种数据事件类型。

<hr>

#### 取消事件绑定

通过`off()`方法可以取消一个事件回调函数的绑定：

```js
ref.off("value", originalCallback);
```

#### 一次性读取数据
在某些场景下，也许需要事件的回调方法只被触发一次，然后立即取消。可以使用`once()`方法：
```js
ref.once("value", function(data) {
  // 执行业务处理，此回调方法只会被调用一次
})
```
<hr>

## 查询数据
Wilddog支持选择性的查询数据。要构造一个查询，需要先指定数据如何排序，可以使用以下几个方法：`orderByChild()`，`orderByKey()`， `orderByValue()`， `orderByPriority()`。接下来，可以组合以下五个方法，构造出一个复杂的条件查询：`limitToFirst()`，`limitToLast()`，`startAt()`，`endAt()`，`equalTo()`。

下面我们来举例如何进行数据查询。假设现在有一些关于恐龙的数据如下：
```json
{
  "lambeosaurus": {
    "height" : 2.1,
    "length" : 12.5,
    "weight": 5000
  },
  "stegosaurus": {
    "height" : 4,
    "length" : 9,
    "weight" : 2500
  }
}
```
**按照指定的子节点排序**
通过将子节点的路径名作为参数传递给`orderByChild()`，可以实现按指定子节点排序。例如，要按照height进行排序，可以：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByChild("height").on("child_added", function(snapshot) {
  console.log(snapshot.key() + " was " + snapshot.val().height + " meters tall");
});
```
不包含指定子节点的数据节点，将会按照该子节点为null进行排序，会排在最前边。

每个查询都只能有一个排序。在一个查询中多次调用`orderByChild()`会抛出错误。

**按照数据节点名称排序**
使用`orderByKey()`方法，可以实现按照数据节点的名称进行排序。下面的例子按照alpha字母顺序读取所有的恐龙数据：
```
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByKey().on("child_added", function(snapshot) {
  console.log(snapshot.key());
});
```

**按照数据节点的值排序**
使用`orderByValue()`方法，我们可以按照子节点的值进行排序。假设恐龙们进行了一场运动会，我们统计到它们的得分数据：

```json
{
  "scores": {
    "bruhathkayosaurus" : 55,
    "lambeosaurus" : 21,
    "linhenykus" : 80,
    "pterodactyl" : 93,
    "stegosaurus" : 5,
    "triceratops" : 22
  }
}
```

要按照得分进行排序，我们可以构造一个这样的查询：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/scores");
ref.orderByValue().on("value", function(snapshot) {
  snapshot.forEach(function(data) {
    console.log("The " + data.key() + " dinosaur's score is " + data.val());
  });
});
```

**按照优先级排序**
关于优先级，请参考API文档中的相关部分。

#### 复杂查询
我们已经了解到数据排序的方法。接下来是limit类的和range类的查询，通过它们，可以构建出更为复杂的查询：

**limit查询**
`limitToFirst()`和`limitToLast()`两个方法用于设置最大多少个子节点数据会被同步。如果我们设置为100，那么初次获取到数据时，就只会最多触发100次事件回调方法。如果数据库中有少于100条消息数据，`child_added`事件将会为每一条消息数据被触发一次。如果消息数量多于100条，那么只有其中的100条会被触发`child_added`事件。如果我们使用了`limitToFirst()`方法，那么被触发的100条将会是排序最前边的100条。如果使用了`limitToLast()`方法，那么被触发的100条将是排序最后的100条。

继续恐龙的例子，我们可以获得体重最大的两种恐龙：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByChild("weight").limitToLast(2).on("child_added", function(snapshot) {
  console.log(snapshot.key());
});
```

我们为`child_added`事件绑定的回调方法只会被执行2次。

同理，我们可以使用`limitToFirst()`方法查询最矮的两种恐龙：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByChild("height").limitToFirst(2).on("child_added", function(snapshot) {
  console.log(snapshot.key());
});
```

我们也可以组合`orderByValue()`方法来使用limit类的查询。如果要构造出恐龙运动会得分的前3名，我们可以构造这样一个查询：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/scores");
scoresRef.orderByValue().limitToLast(3).on("value", function(snapshot) {
  snapshot.forEach(function(data) {
    console.log("The " + data.key() + " dinosaur's score is " + data.val());
  });
});
```

**range查询**
使用`startAt()`，`endAt()`，和`equalTo()`方法，我们可以任意指定任意值的范围进行查询。例如，如果要查询所有至少3米高以上的恐龙，可以组合`orderByChild()`和`startAt()`查询：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByChild("height").startAt(3).on("child_added", function(snapshot) {
  console.log(snapshot.key())
});
```

我们可以使用`endAt()`来查询按照字母排序，所有名字排在Pterodactyl之前的恐龙：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByKey().endAt("pterodactyl").on("child_added", function(snapshot) {
  console.log(snapshot.key());
});
```

注意，`startAt()`和`endAt()`都是包含边界值的，也就是说“pterodactyl”符合上边的查询条件。

我们可以同时使用`startAt()`和`endAt()`来限定一个范围。下面的例子查询出所有名字以字母“b”开头的恐龙：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByKey().startAt("b").endAt("b~").on("child_added", function(snapshot) {
  console.log(snapshot.key());
});
```

这个例子中使用的“~”符号是ASCII中的126字符。因为它排在所有常规的ASCII字符之后，所以这个查询匹配所有以b开头的值。

使用`equalTo()`方法，可以进行精准的查询。例如，查询所有的25米高的恐龙：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.orderByChild("height").equalTo(25).on("child_added", function(snapshot) {
  console.log(snapshot.key());
});
```

同样的，`startAt()`，`endAt()`方法也可以和`orderByValue()`方法组合进行range查询。


**总结**
组合这些方法，我们可以构造出各种复杂的查询。例如，要找出长度小于Stegosaurus但最接近的恐龙的名字：

```js
var ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.child("stegosaurus").child("height").on("value", function(stegosaurusHeightSnapshot) {
  var favoriteDinoHeight = stegosaurusHeightSnapshot.val();
  var queryRef = ref.orderByChild("height").endAt(favoriteDinoHeight).limitToLast(2)
  queryRef.on("value", function(querySnapshot) {
      if (querySnapshot.numChildren() == 2) {
        // 数据是按照height字段的值升序排列的，所以我们需要取得第一个元素。
        querySnapshot.forEach(function(dinoSnapshot) {
          console.log("The dinosaur just shorter than the stegasaurus is " + dinoSnapshot.key());
          // 返回true意味着我们只需要循环forEach()一次
          return true;
        });
      } else {
        console.log("The stegosaurus is the shortest dino");
      }
  });
});
```

<hr>

## 数据规则
本小节介绍在使用各种排序方式时，数据究竟是如何排序的。

**orderByChild**

当使用`orderByChild()`时，包含指定字段的数据将会按照以下规则排序：

- １. 指定字段的值为`null`的子节点数据排在最前边。
- ２. 接下来是指定字段的值为`false`的子节点数据。如果存在多个子节点该字段的值为`false`，那么这些子节点根据节点名按字典序排列。
- ３. 接下来是指定字段的值为`true`的子节点数据。如果存在多个子节点该字段的值为`true`，那么这些子节点根据节点名按字典序排列。
- ４. 接下来是指定字段的值为数值类型的子节点数据，按照升序排列。如果存在多个子节点该指定字段的值相同，那么这些子节点数据按照节点名排序。
- ５. 接下来是字符串类型的值，按照字典序升序排列。如果存在多个子节点该指定字段的值相同，那么这些子节点数据按照节点名排序。
- ６. 最后是对象类型的值，按照节点名升序排列。

**orderByKey**
当使用`orderByKey()`对数据进行排序时，数据将会按照字典顺序排列。注意，节点名只能是字符串类型。

**orderByValue**
当使用`orderByValue()`时，子节点将会按照它们的值进行排序。排序的规则与`orderByChild()`相同，唯一的区别是使用的是本节点的值，而不是节点下指定字段的值。

**orderByPriority**
当使用`orderByPriority()`对数据进行排序时，子节点数据将按照优先级和字段名进行排序。注意，优先级的值只能是数值型或字符串。

- １. 没有优先级的数据（默认）优先。
- ２. 接下来是优先级为数值型的子节点。它们按照优先级数值排序，由小到大。
- ３. 接下来是优先级为字符串的子节点。它们按照优先级的字典序排列。
- ４. 当多个子节点拥有相同的优先级时（包括没有优先级的情况），它们按照节点名排序。节点名可以转换为数值类型的子节点优先（数值排序），接下来是剩余的子节点（字典序排列）。

----
