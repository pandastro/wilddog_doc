/*
Title : 读写数据
Sort : 1
Tmpl : page
*/


## 写入数据

以下几个函数用于写入数据：

`setValue`: 向指定路径写入数据。 

`updateChildValues`: 更新节点的数据。不存在的子节点将会被新增，存在子节点将会被替换。
 
`childByAutoId`: 在数据库中的当前节点下生成一个唯一的ID。任何时刻调用childByAutoId 方法都会生成一个唯一的ID，例如：`messages/users/<unique-user-id>/<username>` 

#### 用 setValue 写数据
setValue 是 Wilddog 最基本的写数据操作。setValue 设置当前节点的值，如果当前节点已经存在值，setValue 会将旧值替换成新值。为了理解setValue 的工作原理，我们创建一个简单博客app，这个博客app的数据存储在这个数据库引用中：

Objective-C 

```
Wilddog *ref = [[Wilddog alloc] initWithUrl:@"https://docs-examples.wilddogio.com/web/saving-data/wildblog"];

```

Swift

```
var ref = Wilddog(url: "https://docs-examples.wilddogio.com/web/saving-data/wildblog")

```


我们先存储一些用户数据。我们为每个用户保存一个唯一的用户名，同时保存全名和出生日期。由于每个用户的用户名都是独一无二的，所以使用setValue方法存储比较好。

首先，我们创建包含用户信息的字典。再创建用户数据节点的引用，然后，使用setValue 方法存储用户的用户名、全名和生日。

Objective-C

```
  NSDictionary *alanisawesome = @{
      @"full_name" : @"Alan Turing",
      @"date_of_birth": @"June 23, 1912"
  };
  NSDictionary *gracehop = @{
      @"full_name" : @"Grace Hopper",
      @"date_of_birth": @"December 9, 1906"
  };
  Wilddog *usersRef = [ref childByAppendingPath: @"users"];
  NSDictionary *users = @{
      @"alanisawesome": alanisawesome,
      @"gracehop": gracehop
  };
  [usersRef setValue: users];

```

Swift

```
  var alanisawesome = ["full_name": "Alan Turing", "date_of_birth": "June 23, 1912"]
  var gracehop = ["full_name": "Grace Hopper", "date_of_birth": "December 9, 1906"]

  var usersRef = ref.childByAppendingPath("users")

  var users = ["alanisawesome": alanisawesome, "gracehop": gracehop]
  usersRef.setValue(users)

```

当我们使用字典将数据保存到数据库中，字典中的元素会自动映射成JSON对象，并保存到指定路径。现在，若我们访问 `https://docs-examples.wilddogio.com/web/saving-data/wildblog/users/alanisawesome/full_name`，将会看到返回值"Alan Turing"。我们也可以直接将数据保存到数据库的指定路径：

Objective-C

```
Wilddog *alanRef = [usersRef childByAppendingPath: @"alanisawesome"];
[alanRef setValue: alanisawesome];

Wilddog *hopperRef = [usersRef childByAppendingPath: @"gracehop"];
[hopperRef setValue: gracehop];

```

Swift

```
usersRef.childByAppendingPath("alanisawesome").setValue(alanisawesome)
usersRef.childByAppendingPath("gracehop").setValue(gracehop)

```

上面介绍的两种保存数据的方式，一种是使用字典将所有数据一次性保存到数据库，另一种是将数据分别保存到数据库的指定路径，但是，最终的效果都是一样的：

```
{
  "users": {
    "alanisawesome": {
      "date_of_birth": "June 23, 1912",
      "full_name": "Alan Turing"
   },
    "gracehop": {
      "date_of_birth": "December 9, 1906",
      "full_name": "Grace Hopper"
    }
  }
}

```

#### 更新已存数据

如果你想同时更新多个子节点，而不覆盖其他的子节点，你可以使用 updateChildValues 方法:

Objective-C

```
Wilddog *hopperRef = [usersRef childByAppendingPath: @"gracehop"];

NSDictionary *nickname = @{
    @"nickname": @"Amazing Grace",
};

[hopperRef updateChildValues: nickname];

```

Swift

```

var hopperRef = usersRef.childByAppendingPath("gracehop")
var nickname = ["nickname": "Amazing Grace"]

hopperRef.updateChildValues(nickname)

```

这样会更新 Grace的数据，更新她的 `nickname` 。如果我们用 `setValue` 而不是 `updateChildValues`，那么`date_of_birth` 和 `full_name` 将都会被删除。



#### 用 childByAutoId 保存列表
当创建数据列表时，我们需要注意应用通常都是多用户的，应根据实际情况调整列表结构。扩展我们上面的实例，假如我们要向博客app中新增posts数据。可能你首先想到的是使用setValue 方法，通过自增的序列方式保存数据，结构如下：

```
// 不推荐用这种方法，建议使用childByAutoId方法!
{
  "posts": {
    "0": {
      "author": "gracehop",
      "title": "Announcing COBOL, a New Programming Language"
    },
    "1": {
      "author": "alanisawesome",
      "title": "The Turing Machine"
    }
  }
}

```

如果用户要在/posts/2节点处增加一个新的post，当只有一个用户操作数据时，这是没有问题的。但是，实际情况是，可能会有多个用户同时添加post。当有两个用户同时向/posts/2节点添加数据时，其中一个posts会被另一个覆盖。

为了解决这个问题，Wilddog提供了childByAutoId 方法，使用该方法每次新加元素时，都会为元素生成一个唯一标识。通过这种方式，多个客户端可以同时向同一个位置添加元素。childByAutoId 生成的唯一标识是基于时间戳计算得来的，所以列表元素是按照时间顺序排列的。 

我们可以通过下面的方式来向博客app写入posts数据：

Objective-C

```
Wilddog *postRef = [ref childByAppendingPath: @"posts"];
NSDictionary *post1 = @{
    @"author": @"gracehop",
    @"title": @"Announcing COBOL, a New Programming Language"
};
Wilddog *post1Ref = [ref childByAutoId];
[post1Ref setValue: post1];

NSDictionary *post2 = @{
    @"author": @"alanisawesome",
    @"title": @"The Turing Machine"
};
Wilddog *post2Ref = [ref childByAutoId];
[post2Ref setValue: post2];

```

Swift

```
let postRef = ref.childByAppendingPath("posts")
let post1 = ["author": "gracehop", "title": "Announcing COBOL, a New Programming Language"]
let post1Ref = postRef.childByAutoId()
post1Ref.setValue(post1)

let post2 = ["author": "alanisawesome", "title": "The Turing Machine"]
let post2Ref = postRef.childByAutoId()
post2Ref.setValue(post2)

```

由于使用了childByAutoId 方法为每个博客post生成了基于时间戳的唯一标识，即使多个用户同时添加博客post也不会产生冲突。Wilddog数据库中的数据结构如下：
```on
{
  "posts": {
    "-JRHTHaIs-jNPLXOQivY": {
      "author": "gracehop",
      "title": "Announcing COBOL, a New Programming Language"
     },
    "-JRHTHaKuITFIhnj02kE": {
      "author": "alanisawesome",
      "title": "The Turing Machine"
    }
  }
}

```

**获取由childByAutoId 生成的唯一ID**
调用 childByAutoId 会返回一个引用，这个引用指向新增数据所在的节点。你可以利用这个ID来获取数据和存储数据

Objective-C

```
NSDictionary *post1 = @{
    @"author": @"gracehop",
    @"title": @"Announcing COBOL, a New Programming Language"
};
Wilddog *post1Ref = [ref childByAutoId];
[post1Ref setValue: post1];

NSString *postId = post1Ref.key;

```

Swift
```
var post1 = ["author": "gracehop", "title": "Announcing COBOL, a New Programming Language"]
var post1Ref = ref.childByAutoId()
post1Ref.setValue(post1)

var postId = post1Ref.key
```

上面例子可以看到，在我们的childByAutoId 引用上调用key 方法就可以获取的唯一ID。

<hr>

## 事件类型
Wilddog 提供了五种数据事件：`Value`，`Child Added`，`Child Changed`，`Child Removed` ，`Child Moved`。可以通过`– observeEventType:withBlock:`方法监听云端的数据变化。

**数据事件 Value**
`Value` 当有数据请求或有任何数据发生变化时触发，用来读取当前节点的数据快照。 `Value` 事件在`– observeEventType:withBlock:` 被调用的时候被触发一次，之后每次数据发生改变都会被触发。当回调函数被调用时候，当前节点下所有数据的快照会被传入作为参数。


**子节点被添加事件 Child Added**
当新的子节点被添加到当前节点，`ChildAdded` 事件被触发，不像`Value` 事件，在 `ChildAdded` 事件的回调函数中传入的参数仅仅是被添加的那个节点的数据静态快照。
如果我们仅仅关心新增节点，我们可以 使用 `ChildAdded`:

Objective-C 

```
// 获取一个我们帖子的引用
Wilddog *ref = [[Wilddog alloc] initWithUrl: @"https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts"];

// 获得新增加的数据
[ref observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {
  NSLog(@"%@", snapshot.value[@"author"]);
  NSLog(@"%@", snapshot.value[@"title"]);
}];

```

Swift

```
// 获取一个我们帖子的引用
var ref = Wilddog(url:"https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts")

// 获得新增加的数据
ref.observeEventType(.ChildAdded, withBlock: { snapshot in
    println(snapshot.value.objectForKey("author"))
    println(snapshot.value.objectForKey("title"))
})
```

**子节点被改变事件 Child Changed**
当子节点发生了改变，`ChildChanged`  事件被触发， `ChildChanged` 的事件回调函数中传入的参数是子节点改变后的数据的静态快照。

Objective-C 

```
// 获取一个我们帖子的引用
Wilddog *ref = [[Wilddog alloc] initWithUrl: @"https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts"];

// 获得变化的数据
[ref observeEventType:WEventTypeChildChanged withBlock:^(WDataSnapshot *snapshot) {
  NSLog(@"The updated post title is %@", snapshot.value[@"title"]);
}];

```

Swift

```
// 获取一个我们帖子的引用
var ref = Wilddog(url:"https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts")

// 获得变化的数据
ref.observeEventType(.ChildChanged, withBlock: { snapshot in
    let title = snapshot.value.objectForKey("title") as? String
    println("The updated post title is \(title)")
})

```


**子节点被删除事件 Child Removed**
当子节点被删除，`ChildRemoved` 事件被触发， `ChildRemoved`与 `ChildAdded`, `ChildChanged` 配合使用，可以覆盖到所有的数据变化。

Objective-C

```
// 获取一个我们帖子的引用
Wilddog *ref = [[Wilddog alloc] initWithUrl: @"https://docs-examples.wildbaseio.com/web/saving-data/wildblog/posts"];

// 获得被删除的数据
[ref observeEventType:WEventTypeChildRemoved withBlock:^(WDataSnapshot *snapshot) {
  NSLog(@"The blog post titled %@ has been deleted", snapshot.value[@"title"]);
}];

```

Swift

```
// 获取一个我们帖子的引用
var ref = Wilddog(url:"https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts")

// 获得被删除的数据
ref.observeEventType(.ChildRemoved, withBlock: { snapshot in
    let title = snapshot.value.objectForKey("title") as? String
    println("The blog post titled \(title)")
})

```


**子节排序发生变化事件 Child Moved**
Child Moved 是当有子节排序发生变化时触发。

<hr>

## 读取数据

让我们重温一下前一篇文章中博客的例子，来理解我们是如何从 Wilddog 数据库中读取数据的。我们的示例应用程序的博客文章是被存储在 url：`https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts `。为了读取数据，我们可以这样做：

Objective-C 

```
// 获取一个我们帖子的引用
Wilddog *ref = [[Wilddog alloc] initWithUrl: @"https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts"];

// 在帖子的引用下，绑定一个 block 去读取数据
[ref observeEventType:WEventTypeValue withBlock:^(WDataSnapshot *snapshot) {
    NSLog(@"%@", snapshot.value);
} withCancelBlock:^(NSError *error) {
    NSLog(@"%@", error.description);
}];

```

Swift

```
// 获取一个我们帖子的引用
var ref = Wilddog(url:"https://docs-examples.wilddogio.com/web/saving-data/wildblog/posts")

// 在帖子的引用下，绑定一个 block 去读取数据
ref.observeEventType(.Value, withBlock: { snapshot in
    println(snapshot.value)
}, withCancelBlock: { error in
    println(error.description)
})
```
如果我们运行这段代码，我们将得到一个包含我们所有帖子的对象。在监听的任一时刻，如果新的数据添加到我们的数据库引用中，这个 block 将会被调用，所以，我们也不需要再编写别的额外代码来实现这个目的。

回调函数传过来一个 WDataSnapshot 对象，WDataSnapshot 对象是在指定的数据路径下，某一个时间点上数据的一个快照。快照的 value 属性返回值是 id 类型。如果没有数据存在于该位置，在快照的值将是 NSNull。

请注意，我们需要指定 WEventType。在我们上面的例子，我们使用的事件类型是 WEventTypeValue 类型。这种方法可以读取一个 Wilddog 数据库的全部内容。Value 是 WEventType 枚举中五种不同的事件类型中的一种，我们可以用这五种枚举类型来读取数据。


<hr>

## 查询数据

我们可以指定多种查询条件，使用相应的查询方法，达到选择性地查询数据的目的。Wilddog 支持选择性的查询数据。构造一个查询，需要先指定数据如何排序，可以使用以下几个方法：`queryOrderedByChild:`，`queryOrderedByKey`， `queryOrderedByValue`， `queryOrderedByPriority`。接下来，可以组合以下五个方法，构造出一个复杂的条件查询：`queryLimitedToFirst:`，`queryLimitedToLast:`，`queryStartingAtValue:`，`queryEndingAtValue:`，`queryEqualToValue:`。

下面我们来举例如何进行数据查询。假设现在有一些关于恐龙的数据如下：

```
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
数据排序的方法有4种：按子节点，节点名，值和优先级。

**按照指定的子节点排序**

通过将子节点的路径名作为参数传递给`queryOrderedByChild:`，可以实现按指定子节点排序。例如，要按照 height 进行排序，可以：

Objective-C

```
Wilddog *ref = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[ref queryOrderedByChild:@"height"]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@ was %@ meters tall", snapshot.key, snapshot.value[@"height"]);
}];

```

Swift

```
let ref = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByChild("height").observeEventType(.ChildAdded, withBlock: { snapshot in
    if let height = snapshot.value["height"] as? Double {
        println("\(snapshot.key) was \(height) meters tall")
    }
})

```

不包含指定子节点的数据节点，将会按照该子节点为 null 进行排序，会排在最前边。

查询还可以通过深度嵌套的子节点进行排序，而不只是只有一个子节点的级别。如果你有这样深度嵌套的数据，也可以实现排序：

```
{
  "lambeosaurus": {
    "dimensions": {
      "height" : 2.1,
      "length" : 12.5,
      "weight": 5000
    }
  },
  "stegosaurus": {
    "dimensions": {
      "height" : 4,
      "length" : 9,
      "weight" : 2500
    }
  }
}

```

现在要查询 height，我们使用完整路径，而不是一个单一的 key：

Objective-C

```
Wilddog *ref = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[ref queryOrderedByChild:@"dimensions/height"]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@ was %@ meters tall", snapshot.key, snapshot.value[@"height"]);
}];

```

Swift

```
let ref = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByChild("dimensions/height").observeEventType(.ChildAdded, withBlock: { snapshot in
    if let height = snapshot.value["height"] as? Double {
        println("\(snapshot.key) was \(height) meters tall")
    }
})

```

每个查询都只能有一个排序。在一个查询中多次调用`queryOrderedByChild:`会抛出错误。

**按照数据节点名称排序**

使用`queryOrderedByKey`函数，可以实现按照数据节点的名称进行排序。下面的例子按照 alpha 字母顺序读取所有的恐龙数据：

Objective-C

```
Wilddog *ref = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[ref queryOrderedByKey]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@ was %@", snapshot.key, snapshot.value[@"height"]);
}];

```

Swift

```
let ref = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByKey().observeEventType(.ChildAdded, withBlock: { snapshot in
    if let height = snapshot.value["height"] as? Double {
        println("\(snapshot.key) was \(height)")
    }
})

```

**按照数据节点的值排序**
使用`queryOrderedByValue`方法，我们可以按照子节点的值进行排序。假设恐龙们进行了一场运动会，我们统计到它们的得分数据：

```
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

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/scores"];
[[scoresRef queryOrderedByValue] observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {
    NSLog(@"The %@ dinosaur's score is %@", snapshot.key, snapshot.value);
}];

```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/scores")
scoresRef.queryOrderedByValue().observeEventType(.ChildAdded, withBlock: { snapshot in
    if let score = snapshot.value as? Int {
        println("The \(snapshot.key) dinosaur's score is \(score)")
    }
})

```

**按照优先级排序**

关于优先级，请参考 API 文档中的相关部分。

#### 复杂查询

我们已经了解到数据排序的方法。接下来是 limit 类的和 range 类的查询，通过它们，可以构建出更为复杂的查询：

**limit查询**

`queryLimitedToFirst:`和`queryLimitedToLast: `两个方法用于设置最大多少个子节点数据会被同步。如果我们设置为100，那么初次获取到数据时，就只会最多触发100次事件回调方法。如果数据库中少于100条消息数据，`ChildAdded`事件将会为每一条消息数据被触发一次。如果消息数量多于100条，那么只有其中的100条会被触发`ChildAdded`事件。如果我们使用了`queryLimitedToFirst:`方法，那么被触发的100条将会是排序最前边的100条。如果使用了`queryLimitedToLast:`方法，那么被触发的100条将是排序最后的100条。

继续恐龙的例子，我们可以获得体重最大的两种恐龙：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[[ref queryOrderedByChild:@"weight"] queryLimitedToLast:2] observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@", snapshot.key);
}];

```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByChild("weight").queryLimitedToLast(2)
   .observeEventType(.ChildAdded, withBlock: { snapshot in
    println(snapshot.key)
})

```

我们为`ChildAdded`事件绑定的回调方法只会被执行2次。

同理，我们可以使用`queryLimitedToFirst:`方法查询最矮的两种恐龙：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[[ref queryOrderedByChild:@"height"] queryLimitedToFirst:2]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@", snapshot.key);
}];

```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByChild("height").queryLimitedToFirst(2)
   .observeEventType(.ChildAdded, withBlock: { snapshot in
    println(snapshot.key)
})

```

我们也可以组合`queryOrderedByValue`方法来使用 limit 类的查询。如果要构造出恐龙运动会得分的前3名，我们可以构造这样一个查询：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/scores"];
[[[scoresRef queryOrderedByValue] queryLimitedToLast:3]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"The %@ dinosaur's score is %@", snapshot.key, snapshot.value);
}];

```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/scores")
scoresRef.queryOrderedByValue().queryLimitedToLast(3).observeEventType(.ChildAdded, withBlock: { snapshot in

    println("The \(snapshot.key) dinosaur's score is \(snapshot.value)")
})

```

**range查询**

使用`queryStartingAtValue:`，`queryEndingAtValue:`和`queryEqualToValue:`方法，可以为我们的查询指定任意的起止范围。例如，如果要查询所有3米高以上的恐龙，可以组合`queryOrderByChild:`和`queryStartingAtValue:`查询：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[[ref queryOrderedByChild:@"height"] queryStartingAtValue:@3]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@", snapshot.key);
}];

```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByChild("height").queryStartingAtValue(3)
   .observeEventType(.ChildAdded, withBlock: { snapshot in

    println(snapshot.key)
})

```

按照字母排序，我们可以使用`queryEndingAtValue:`来查询所有名字排在 Pterodactyl 之前的恐龙：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[[ref queryOrderedByKey] queryEndingAtValue:@"pterodactyl"]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@", snapshot.key);
}];

```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByKey().queryEndingAtValue("pterodactyl")
   .observeEventType(.ChildAdded, withBlock: { snapshot in

    println(snapshot.key)
})

```

注意，`queryStartingAtValue:`和`queryEndingAtValue:`是包含边界值的，也就是说“pterodactyl”符合上边的查询条件。

我们可以同时使用`queryStartingAtValue:`和`queryStartingAtValue:`来限定一个范围。下面的例子查询出所有名字以字母“b”开头的恐龙：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[[[ref queryOrderedByKey] queryStartingAtValue:@"b"] queryEndingAtValue:@"b\uf8ff"]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@", snapshot.key);
}];

```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByKey().queryStartingAtValue("b").queryEndingAtValue("b\u{f8ff}")
   .observeEventType(.ChildAdded, withBlock: { snapshot in

    println(snapshot.key)
})

```
这个例子中使用的“~”符号是 ASCII 中的126字符。因为它排在所有常规的 ASCII 字符之后，所以这个查询匹配所有以b开头的值。

使用`queryEqualToValue:`函数，可以进行精准的查询。例如，查询所有的25米高的恐龙：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[[ref queryOrderedByChild:@"height"] queryEqualToValue:@25]
    observeEventType:WEventTypeChildAdded withBlock:^(WDataSnapshot *snapshot) {

    NSLog(@"%@", snapshot.key);
}];
```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.queryOrderedByChild("height").queryEqualToValue(25)
   .observeEventType(.ChildAdded, withBlock: { snapshot in

    println(snapshot.key)
})

```

**总结**
组合这些函数，我们可以构造出各种复杂的查询。例如，要找出长度小于 Stegosaurus 但最接近的恐龙的名字：

Objective-C

```
Wilddog *scoresRef = [[Wilddog alloc] initWithUrl:@"https://dinosaur-facts.wilddogio.com/dinosaurs"];
[[[ref childByAppendingPath:@"stegosaurus"] childByAppendingPath:@"height"]
 observeEventType:WEventTypeValue withBlock:^(WDataSnapshot *stegosaurusHeightSnapshot) {
     NSNumber *favoriteDinoHeight = stegosaurusHeightSnapshot.value;
     WQuery *queryRef = [[[ref queryOrderedByChild:@"height"] queryEndingAtValue:favoriteDinoHeight] queryLimitedToLast:2];
     [queryRef observeSingleEventOfType:WEventTypeValue withBlock:^(WDataSnapshot *querySnapshot) {
         if (querySnapshot.childrenCount == 2) {
             for (WDataSnapshot* child in querySnapshot.children) {
                 NSLog(@"The dinosaur just shorter than the stegasaurus is %@", child.key);
                 break;
             }
         } else {
             NSLog(@"The stegosaurus is the shortest dino");
         }
     }];
 }];
 
```

Swift

```
let scoresRef = Wilddog(url:"https://dinosaur-facts.wilddogio.com/dinosaurs")
ref.childByAppendingPath("stegosaurus").childByAppendingPath("height")
    .observeEventType(.Value, withBlock: { stegosaurusHeightSnapshot in
        if let favoriteDinoHeight = stegosaurusHeightSnapshot.value as? Double {
            let queryRef = ref.queryOrderedByChild("height").queryEndingAtValue(favoriteDinoHeight).queryLimitedToLast(2)
            queryRef.observeSingleEventOfType(.Value, withBlock: { querySnapshot in
                if querySnapshot.childrenCount == 2 {
                    let child: WDataSnapshot = querySnapshot.children.nextObject() as WDataSnapshot
                    println("The dinosaur just shorter than the stegasaurus is \(child.key)");
                } else {
                    println("The stegosaurus is the shortest dino");
                }
            })
        }
    })

```

## 排序规则
本小节介绍在使用各种排序方式时，数据究竟是如何排序的。

**queryOrderedByChild**

当使用`queryOrderedByChild:`时，按照子节点的公有属性 key 的 value 进行排序。仅当 value 为单一的数据类型时，排序有意义。如果 key 属性有多种数据类型时，排序不固定，此时不建议使用`queryOrderedByChild:`获取全量数据，例如，

```json
{
  "scores": {
    "no1" : {
        "name" : "tyrannosaurus",
        "score" : "120"
    },
    "no2" : {
        "name" : "bruhathkayosaurus",
        "score" : 55
    },
    "no3" : {
        "name" : "lambeosaurus",
        "score" : 21
    },
    "no4" : {
        "name" : "linhenykus",
        "score" : 80
    }, 
    "no5" : {
        "name" : "pterodactyl",
        "score" : 93
    }, 
    "no6" : {
        "name" : "stegosaurus",
        "score" : 5
    }, 
    "no7" : {
        "name" : "triceratops",
        "score" : 22
    }, 
    "no8" : {
        "name" : "brontosaurus",
        "score" : true
    }
  }
}

```

霸王龙的分数是`NString`类型，雷龙的分数是`BOOL`类型，而其他恐龙的分数是`NSNumber`类型，此时使用`queryOrderedByChild:`获得全量数据时，是一个看似固定的排序结果；但是配合使用`queryLimitedToFirst:`时，将获得不确定的结果。`NSObject`类型数据的 value 值为 null，不会出现在结果中。
当配合使用`queryStartingAtValue:`、`queryEndingAtValue:`和`queryEqualToValue:`时，如果子节点的公有属性 key 包含多种数据类型，将按照这些函数的参数的类型排序，即只能返回这个类型的有序数据。上面的数据如果使用 `[[[ref queryOrderedByChild:@"score"]queryStartingAtValue:@60]queryLimitedToFirst:4]` 将得到下面的结果：

```json
{
   "no4" : {
       "name" : "linhenykus",
       "score" : 80
   },
   "no5" : {
       "name" : "pterodactyl",
       "score" : 93
   }
}
  
```

<p style='color:red'><em>注意：如果 path 与 value 的总长度超过1000字节时，使用 `queryOrderedByChild:` 将搜索不到该数据。</em></p>

**queryOrderedByKey**

当使用`queryOrderedByKey`对数据进行排序时，数据将会按照字典顺序排列。注意，节点名只能是字符串类型。

**queryOrderedByValue**

当使用`queryOrderedByValue`时，按照直接子节点的 value 进行排序。仅当 value 为单一的数据类型时，排序有意义。如果子节点包含多种数据类型时，排序不固定，此时不建议使用`queryOrderedByValue`获取全量数据，例如，

```json
{
  "scores": {
    "tyrannosaurus" : "120",
    "bruhathkayosaurus" : 55,
    "lambeosaurus" : 21,
    "linhenykus" : 80,
    "pterodactyl" : 93,
    "stegosaurus" : 5,
    "triceratops" : 22,
    "brontosaurus" : true
  }
}

```

霸王龙的分数是 `NSString`类型，雷龙的分数是 `BOOL` 类型，而其他恐龙的分数是 `NSNumber` 类型，此时使用 `queryOrderedByValue` 获得全量数据时，是一个看似固定的排序结果；但是配合使用`queryLimitedToFirst:`时，将获得不确定的结果。`NSObject`类型数据的 value 值为 null，不会出现在结果中。
当配合使用`queryStartingAtValue:`、`queryEndingAtValue:`和`queryEqualToValue:`时，如果子节点的 value 包含多种数据类型，将按照这些函数的参数的类型排序，即只能返回这个类型的有序数据。上面的数据如果使用`[[[ref queryOrderedByValue]queryStartingAtValue:@60]queryLimitedToFirst:4]`将得到下面的结果：
```json
{
    "linhenykus" : 80,
    "pterodactyl" : 93
}
```
<p style='color:red'><em>注意：如果 path 与 value 的总长度超过1000字节时，使用`queryOrderedByValue:`将搜索不到该数据。</em></p>

**queryOrderedByPriority**

当使用`queryOrderedByPriority`对数据进行排序时，子节点数据将按照优先级和字段名进行排序。注意，优先级的值只能是数值型或字符串。

1, 没有优先级的数据（默认）优先。

2, 接下来是优先级为数值型的子节点。它们按照优先级数值排序，由小到大。

3, 接下来是优先级为字符串的子节点。它们按照优先级的字典序排列。

4, 当多个子节点拥有相同的优先级时（包括没有优先级的情况），它们按照节点名排序。节点名可以转换为数值类型的子节点优先（数值排序），接下来是剩余的子节点（字典序排列）。

