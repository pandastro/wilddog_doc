/*
Title : 开发向导
Sort : 1
Tmpl : page
*/

## 1.安装与配置

### 创建一个账号

首先，注册并登录Wilddog账号，进入[控制面板](https://www.wilddog.com/dashboard)。然后，在控制面板中，添加一个新的应用。
你会得到一个应用的URL `https://<appId>.wilddogio.com/`。你可以把这个URL理解为云端数据库的地址。

### 使用Rest API

在任何支持HTTPS的平台和语言中，都可以使用Wilddog Rest API。在本向导中，我们使用[cURL](https://en.wikipedia.org/wiki/CURL)命令行工具发送HTTPS请求。

<br>
## 2.了解数据

### 数据是一棵 JSON 树

所有的数据都存储在各个 JSON 对象中，没有任何表的概念。当你把数据添加到这棵json 树中，这些数据就变成这棵树的子树。比如，我们在`users/mchen` 下增加 `widget`后，我们的数据是这样的:

``` json
{
  "users": {
    "mchen": {
      "friends": { "brinchen": true },
      "name": "Mary Chen",
      // 新数据节点会增加在已经存在的JSON树中
      "widgets": { "one": true, "three": true }
    },
    "brinchen": { ... },
    "hmadi": { ... }
  }
}
```

### 使用数据URL

通过REST API读取和写入数据时，我们在`curl`请求中包含一个指向数据地址的URL，这个URL会指向我们存储的所有数据。在这个示例中，我们将使用`https://docs-examples.wilddogio.com/web/data`这个地址。

也可以直接访问子节点的数据，例如，要访问Mary Chen的name，只需要在URL之后追加`users/mchen/name`即可。

```
curl https://docs-examples.wilddogio.com/web/data/users/mchen/name.json
```

<!--
####Wilddog中的数组
Wilddog对数组没有原生的支持，如果要存储数组，实际上存储的是以下标数字作为key的对象。

```
// 我们发送的数据

['hello', 'world']

// wilddog存储的数据

{0: 'hello', 1: 'world'}

//然而，为了帮助用户存储数组，当数据通过val()或者REST API读取时，如果数据看起来像数组，Wilddog就会把数据转换为数组。特别地，如果数据的key值都是整数，并且超过一半是在0到最大值之间并且没有空值，那么Wilddog会将其转换为数组。这一部分要引起注意：

// 我们发送的数据

['a', 'b', 'c', 'd', 'e']

// wilddog保存的数据
{0: 'a', 1: 'b', 2: 'c', 3: 'd', 4: 'e'}

// 由于key是数字型，且是有序的
// 如果我们查询数据，会得到以下形式
['a', 'b', 'c', 'd', 'e']

// 但是，如果我们删除a、b、d
// 它们不再是有序的
// 我们不会再得到数组形式
{2: 'c', 4: 'e'}

```
当前没有办法改变或者阻止这种行为，理解这一点可以知道存储类似数组数据时什么能做什么不能做。
为什么不提供对数组的全支持呢？这是因为数组的索引不是永久的，不是id唯一的，并发编辑时是容易出现问题。
例如，假设两个用户通过远程服务器同时更新一个数组，用户A想要改变key为2的数值，用户B想要移动并改变它，那结果会出现错误。
下面是其中的一种情况：
```
// 初始数据

['a', 'b', 'c', 'd', 'e']

// 用户A将key为2的记录移动到位置5

// 用户B要删除key为2的记录

// 用户C将key为2的记录更新为foo

// 最理想的是以下的情况

['a', 'b', 'd', 'e']

// 实际发生的是以下的情况

['a', 'c', 'foo', 'b']

```
那什么时候可以使用数组呢？如果满足以下条件，可以在Wilddog中存储数组：
* 每次保证只有一个客户端写数据
* 要删除元素的时候，我们重新保存整个数组，而不是调用`.remove()`方法
* 特别注意根据数组索引更新内容的情况

-->

### 限制和约束

| 描述 | 约束 | 备注 |
| --- | --- | --- |
| 树的深度 |32 | |
|key的长度 | 768bytes | UTF-8 编码，不能包含`.` `$` `#` `[` `]` `/`和 ASCII控制字符0-31和127 |
| 一个叶子节点的数据大小 | 1mb | UTF-8 编码 |
| 通过SDK写入的数据大小限制 | 2mb | UTF-8 编码 |
| 通过 REST 写入数据大小限制 |4mb | &nbsp; |
| 一次能读取的节点 |2000 | &nbsp; |
| 一次条件查询能返回的最大条数 |500 | 如使用 limitToFirst、limitToLast等 |

### 特殊字符转义

当节点的key值是特殊字符时可能导致发送的curl请求URL无效，此时我们需要将特殊字符转义

| 特殊字符 | 转义符 |
| --- | --- |
|  ? | %3F  
|  % | %25  
|  + | %2B 
|  = | %3D  
|  & | %26 
<br>
## 3.保存数据

### 保存数据的方式

| method | 说明 | 
| --- | --- | 
| PUT	|	写入或更新路径下的数据，如 `messages/users/user1/<data>`
| PATCH	|	更新指定路径下的部分key值，但是不影响其他的数据
| POST	|	在Wilddog数据库中增加一个节点，每次发送一个POST请求都会生成一个独一无二的ID，如`messages/users/<unique-id>/<data>`		| 
| DELETE	|	删除指定数据路径下的数据		| 

### 使用PUT来写入数据

REST API基本的写入数据的操作是`PUT`。为了演示数据存储，我们将建立一个博客应用，应用的所有数据都存储在Wilddog应用对应的URl`https://docs-examples.wilddogio.com/rest/saving-data/wildblog`中。

下面来存储一些用户的数据到数据库中，我们存储每个用户的唯一用户名，还存储全名和出生日期。由于用户名是独一无二的，所以适合使用`PUT`而不是`POST`方法，因为我们已经有作为key值的字段，不需要生成。

使用`PUT`方法，我们可以写入`string`, `number`, `boolean`,`array`或者任意的JSON对象到我们的数据库，这里我们将传递一个JSON对象：
```
curl -X PUT -d '{
  "alanisawesome": {
    "name": "Alan Turing",
    "birthday": "June 23, 1912"
  }
}' 'https://docs-examples.wilddogio.com/rest/saving-data/wildblog/users.json'
```

当一个JSON对象被存储到数据库中，对象的属性被自动映射到指定位置。如果我们定位到新添加的节点上，我们会看到值“Alan Turing”，我们也可以直接保存数据到子路径节点上：
```
curl -X PUT -d '"Alan Turing"' \
  'https://docs-examples.wilddogio.com/rest/saving-data/wildblog/users/alanisawesome/name.json'
```
```
curl -X PUT -d '"June 23, 1912"' \
  'https://docs-examples.wilddogio.com/rest/saving-data/wildblog/users/alanisawesome/birthday.json'
```

以上两个例子采用了两种方法写入数据：一次性写入一个JSON对象，和分别写入每个子数据节点。两种方式最终写入数据的结果是一样的：

```json
{
  "users": {
    "alanisawesome": {
      "date_of_birth": "June 23, 1912",
      "full_name": "Alan Turing"
    }
  }
}
```

成功的请求将返回HTTP 200 OK状态码，并且响应中会包含存储到数据库中的数据。上面的两个例子中，对于关注数据变化的其它客户端，第一个例子的做法仅仅触发一个事件，而第二个例子将触发两个事件。注意，如果要写入的路径节点下已经存在数据，第一种方法会覆盖已有的数据，而第二种方法只是修改两个子节点的值，而不会影响已有的其他子节点的数据。`PUT`方法等同于JavaScript SDK的`set()`方法。

### 使用PATCH来更新数据
使用`PATCH`请求，我们可以更新指定子节点的数据，而不覆盖其它已经存在的数据。例如使用`PATCH`请求为Turing添加nickname：

```
curl -X PATCH -d '{
  "nickname": "ACE"
}' \
  'https://docs-examples.wilddogio.com/rest/saving-data/users/alanisawesome.json'
```

上面的请求将`nickname`添加到`alanisawesome`对象而不删除子节点`name`和`birthday`。如果使用的是`PUT`请求，`name`和`birthday`将会被删除，因为他们没有出现在请求中。执行完`PATCH`请求之后，数据库中的数据如下：

```json
{
  "users": {
    "alanisawesome": {
      "date_of_birth": "June 23, 1912",
      "full_name": "Alan Turing",
      "nickname": "ACE"
    }
  }
}
```

成功的请求将返回HTTP 200 OK状态码，并且响应中会包含更新到数据库中的数据。

### 保存列表数据

如果要给添加到数据库中的元素生成一个独一无二的、基于时间戳的key，我们可以使用`POST`请求。对于我们的`users`路径，我们自行定义user数据的key是很有必要的，因为每个用户都有有唯一用户名。但是当用户发表博客时，我们可以使用`POST`请求为博客数据自动生成key。

```
curl -X POST -d '{
  "author": "alanisawesome",
  "title": "The Turing Machine"
}' 'https://docs-examples.wilddogio.com/rest/saving-data/wildblog/posts.json'
```
我们的`posts`路径下的数据将会是这样：

```json
{
  "posts": {
    "-JRHTHaKuITFIhnj02kE": {
      "author": "alanisawesome",
      "title": "The Turing Machine"
    }
  }
}
```

注意，`key-JRHTHaKuITFIhnj02kE`是自动生成的，因为我们使用的是`POST`请求。成功的请求将返回HTTP 200 OK状态码，并且响应中会包含新数据的key。

```
{"name":"-JRHTHaKuITFIhnj02kE"}
```

### 删除数据

要删除Wilddog中的数据，我们可以发送`DELETE`请求到要删除数据对应的路径上。下面的命令将删除`users`路径上的`alanisawesome`：

```
curl -X DELETE \
  'https://docs-examples.wilddogio.com/rest/saving-data/users/alanisawesome.json'
```

成功的请求将返回HTTP 200 OK状态码，和一个空的JSON。

### URI参数

当向数据库写入数据时，REST API可以接受以下的参数：

**auth**

`auth`参数允许访问受Wilddog规则表达式保护的数据，并且支持所有的请求方式。auth参数的值可以是Wilddog应用的超级密钥，也可以是一个认证token。下面的例子中我们发送一个包含`auth`参数的`POST`请求，参数的值CREDENTIAL可以是超级密钥，也可以是认证token。

```
curl -X POST -d '{"Authenticated POST request"}' \
  'https://docs-examples.wilddogio.com/rest/saving-data/auth-example.json?auth=CREDENTIAL'
```

**print**

`print`参数让我们可以设置响应的内容格式。把`print=pretty`添加到请求中将会返回易读的格式。`GET`、`PUT`、`POST`和`PATCH`请求都支持`print=pretty`参数。


#### 写入服务端数据

服务端数值使用占位符： `.sv`。`.sv`的值就是我们期望的服务端数值类型。例如，当一个用户被创建的时候需要设置一个时间戳，我们应该如下操作：
```
curl -X PUT -d '{".sv": "timestamp"}' \
  'https://docs-examples.wilddogio.com/rest/saving-data/alanisawesome/createdAt.json'
```
服务端数值现在只支持时间戳，关于unix时间戳的百科，请参考[百科](http://baike.baidu.com/link?url=VQMFk3ej6ORZFtAhKYF5P6ow_p1XqZ5RgzFHNQFJNgc5U_DCT4nH6MVXkIvSmvO5gLP5DrB7ZsrnZc-2cT5bHa)。

#### 提高写入性能

如果我们要向数据库写入大量的数据，我们可以使用`print=silent`参数来提高写入性能和减少带宽占用。在正常的写操作中，服务器使用已经写入到数据库中的JSON数据来响应。当指定了`print=slient`参数，服务器会在数据传输完后立即关闭连接来减少带宽占用。

如果我们需要发送大量的请求到数据库，我们可以在HTTPS请求头中添加`Keep-Alive`请求来复用连接。

### 错误码

REST API将在以下情况返回错误码：

 |  错误码  |  描述   | 
 | ----    |    ---- | 
 | 404 Not Found		|	通过HTTP请求而不是HTTPS请求 | 
 | 400 Bad Request		|	不能解析PUT或POST数据；丢失PUT或POST数据；PUT或POST数据过长；REST API调用路径中包含非法的子节点名字 | 
 | 417 Expectation Failed	|	REST API调用没有指定Wilddog应用id | 
 | 403 Forbidden		|	请求违反规则表达式的约束 | 
<br>
## 4.获取数据

### 使用GET方法读取数据

我们可以发送`GET`请求到数据的URL来读取数据，让我们继续博客的示例，读取全部的博客数据。

```
curl 'https://docs-examples.wilddogio.com/rest/saving-data/wdblog/posts.json?print=pretty'
```

成功的请求将返回HTTP 200 OK状态码，并且响应中会包含读取到的数据。

### 添加URI参数

当我们从数据库中读取数据的时候，REST API可以接受多个参数。下面是最常用的参数。想了解全部的参数，请参见 [REST API文档](/rest/api)。

**auth**
`auth`参数允许访问受Wilddog规则表达式保护的数据，并且支持所有的请求方式。`auth`参数的值可以是Wilddog应用的超级密钥，也可以是一个认证token。下面的例子中我们发送一个包含`auth`参数的`POST`请求，参数的值CREDENTIAL可以是超级密钥，也可以是认证token。

```
curl -X POST -d '{"Authenticated POST request"}' \
  'https://docs-examples.wilddogio.com/rest/saving-data/auth-example.json?auth=CREDENTIAL'
```

**print**
指定`print=pretty`返回易读格式的数据。
```s
curl 'https://<appId>.wilddogio.com/users/jack/name.json?print=pretty'
```
指定`print=silent`返回204 No Content状态码
```
curl 'https://<appId>.wilddogio.com/users/jack/name.json?print=silent'
```

**callback**
为了让来自web客户端的rest请求实现跨域，你可以用JSONP将响应封装到JavaScript回调方法中。使用 `callback=` 让REST API将返回的数据封装到你指定的回调方法中。
```
<script>
  function gotData(data) {
    console.log(data);
  }
</script>
<script src="https://<appId>.wilddogio.com/.json?callback=gotData"></script>
```

**shallow**
这是一个高级功能，目标是帮助处理大的数据集而不下载数据集的全部。设置 `shallow=true` 将限制数据返回的深度。如果返回的数据是JSON原始类型 (如string, number 或 boolean)， 它的value将被返回。 如果数据的snapshot是 JSON 对象，每一个key的value都将被截断成布尔类型`true`。
```
{
  "message": {
    "user": {
      "name": "Chris"
    },
    "body": "Hello!"
  }
}

// REST请求路径为 /message.json?shallow=true
// 返回的数据如下:
{
  "user": true,
  "body": true
}

// REST请求路径为 /message/body.json?shallow=true
// 返回的数据如下:
"Hello!"
```

使用shallow参数后， 将不能使用其他查询参数。

#### 读取服务端时间戳
当我们需要获取服务器的当前时间戳时，可以进行如下操作:
```
curl 'https://<appId>.wilddogio.com/.json?sv=timestamp'
```
服务端数值现在只支持时间戳，关于unix时间戳的百科，请参考[百科](http://baike.baidu.com/link?url=VQMFk3ej6ORZFtAhKYF5P6ow_p1XqZ5RgzFHNQFJNgc5U_DCT4nH6MVXkIvSmvO5gLP5DrB7ZsrnZc-2cT5bHa)。

### 查询数据
你可以使用一些参数来对数据进行查询。首先使用`orderBy`参数来指定数据的排列顺序，然后结合使用`limitToFirst`，`limitToLast`，`startAt`，`endAt`和`equalTo`参数来对数据进行查询。

注意：使用REST API对数据进行查询时，返回的数据集没有固定的顺序，这是因为JSON解释器没有强制排序。如果需要排序，你需要自己在程序中对返回的数据进行排序。

下面我们来举例如何进行数据查询。假设现在有一些关于恐龙的数据如下：
```
{
  "lambeosaurus": {
    "height": 2.1,
    "length": 12.5,
    "weight": 5000
  },
  "stegosaurus": {
    "height": 4,
    "length": 9,
    "weight": 2500
  }
}
```
有四种方式对数据进行查询：按照子节点的value值，按照节点名称key，按照节点value值和节点优先级priority。查询条件以其中一个参数开头，后面必须与`limitToFirst`，`limitToLast`，`startAt`，`endAt`和`equalTo`这些参数配合使用。

**按照子节点value值排序**
将子节点的名称传给`orderBy`参数，就可以按照指定子节点的value值进行排序。
例如，想要查询所有的高度大于等于3米的恐龙，我们可以使用以下参数查询：
```
curl 'https://dinosaur-facts.wilddogio.com/dinosaurs.json?orderBy="height"&startAt=3&print=pretty'
```

注意：
如果你想要在应用中按照某个子节点的value值排序，你需要在规则表达式中配置`".indexOn":"<childkey>"`。详细的文档请参考规则表达式API文档中关于 [.indexOn](/rule/api#-indexOn0) 的介绍。

**按照数据节点名称排序**
可以通过使用`orderBy="$key"`参数来按照节点名称查询数据。
下面的例子将返回名称在a到m之间所有的节点：
```
curl 'https://dinosaur-facts.wilddogio.com/dinosaurs.json?orderBy="$key"&startAt="a"&endAt="m"&print=pretty'
```

**按照节点的value值排序**
我们可以按照节点的value值进行排序。
例如，恐龙举办运动会，我们按照以下格式记录恐龙的成绩：
```
{
  "scores": {
    "bruhathkayosaurus": 55,
    "lambeosaurus": 21,
    "linhenykus": 80,
    "pterodactyl": 93,
    "stegosaurus": 5,
    "triceratops": 22
  }
}
```
要查询所有成绩高于50分的恐龙，我们可以使用以下参数查询：
```
curl 'https://dinosaur-facts.wilddogio.com/scores.json?orderBy="$value"&startAt=50&print=pretty'
```
参考数据排序部分的介绍，了解使用`orderBy="$value"`参数查询时，数据是如何排序的。

注意：
如果你想要在应用中按照节点的value值排序，你需要在规则表达式中配置`".indexOn" : ".value"`。详细的文档请参考规则表达式API文档中关于 [.indexOn](https://z.wilddog.com/rule/api#-indexOn0) 的介绍。

**按照优先级排序**
可以使用`orderBy="$priority"`参数来按照节点的优先级查询数据。详细的优先级介绍请参见 [API文档](https://z.wilddog.com/rest/api#Priorities0)。

### 复杂查询

可以将多个参数进行组合使用，实现更复杂的查询功能。

**limit查询**
`limitToFirst`和`limitToLast`参数用来设置返回结果集的大小。例如，查询条件限制为100，如果记录数小于100，则会返回所有的数据，如果记录数超过100，则会返回其中的100条记录，使用`limitToFirst`得到最前面的100条记录，使用`limitToLast`得到最后面的100条记录。

在恐龙数据库中使用`orderBy`和`limitToLast`，我们可以得到体重最重的两头恐龙的信息：
```
curl 'https://dinosaur-facts.wilddogio.com/dinosaurs.json?orderBy="weight"&limitToLast=2&print=pretty'
```
同样，我们可以使用`limitToFirst`参数来得到高度最小的两头恐龙的信息：
```
curl 'https://dinosaur-facts.wilddogio.com/dinosaurs.json?orderBy="height"&limitToFirst=2&print=pretty'
```
我们可以将`limitToLast`参数与`orderBy="$value"`参数配合使用。例如，我们给恐龙运动会创建一个排行榜，查询分数最高的三头恐龙，则可以使用以下参数查询：
```
curl 'https://dinosaur-facts.wilddogio.com/scores.json?orderBy="$value"&limitToLast=3&print=pretty'
```

**range查询**
使用`startAt`，`endAt`和`equalTo`参数来设置查询范围的起点和终点。例如，我们想要查询至少3米高的恐龙，可以使用`orderBy`和`startAt`的组合查询：
```
curl 'https://dinosaur-facts.wilddogio.com/dinosaurs.json?orderBy="height"&startAt=3&print=pretty'
```

我们可以使用`endAt`参数来查询名字的字典顺序在Pterodactyl之前的所有的恐龙：
```
curl 'https://dinosaur-facts.wilddogio.com/dinosaurs.json?orderBy="$key"&endAt="pterodactyl"&print=pretty'
```

我们可以使用`startAt`和`endAt`的组合来控制我们查询的范围。下面的例子是查询名字以b开头的所有的恐龙：
```
curl 'https://dinosaur-facts.wilddogio.com/dinosaurs.json?orderBy="$key"&startAt="b"&endAt="b~"&print=pretty'
```
波浪线在ASCII中的编码是126，它排在ASCII中常规字符的后面，查询返回的是所有b开头的字符串。


范围查询在分页中非常有用。


### 数据排序

这一部分主要介绍在使用各种排序方式时，数据究竟是如何排序的。 

**orderBy**
当使用`orderBy`参数时，按照子节点的公有属性key的value进行排序。仅当value为单一的数据类型时，排序有意义。如果key属性有多种数据类型，则排序不固定，此时不建议使用`orderBy`参数获取全量数据。例如，
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
霸王龙的分数是`string`类型，雷龙的分数是`boolean`类型，而其他恐龙的分数是`numberic`类型，此时使用 `orderBy`参数获得全量数据，返回的是一个看似固定的排序结果；但是配合使用`limitToFirst`或`limitToLast`时，将获得不确定的结果。

当配合使用`startAt`、`endAt`和`equalTo`参数时，如果子节点的公有属性key包含多种数据类型，将按照参数的类型排序，即只能返回这个类型的有序数据。 上面的数据如果使用 `orderBy="score"&startAt=60` 将得到下面的结果：
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
 `Object`类型数据的 value 值为 null，不会出现在结果中。
 <p style='color:red'><em>注意：如果path与value的总长度超过1000字节时，使用`orderBy`参数将搜索不到该数据。</em></p>

**orderBy="$key"**
当使用`orderBy="$key"`参数对数据进行排序时，数据将会按照key值增序排列。注意，key值只能是字符型。
1. 1.key值能够被解析成数字的节点排在最前面，增序排列
2. 2.接下来是字符型key值，按照字典顺序增序排列

**orderBy="$value"**
当使用`orderBy="$value"`参数时，按照直接子节点的 value 进行排序。仅当 value 为单一的数据类型时，排序有意义。如果子节点包含多种数据类型，则排序不固定， 此时不建议使用`orderBy="$value"`获取全量数据，例如，
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
霸王龙的分数是`string`类型，雷龙的分数是`boolean`类型，而其他恐龙的分数是`numberic`类型，此时使用 `orderBy="$value"`参数获得全量数据，返回的是一个看似固定的排序结果；但是配合使用`limitToFirst`或`limitToLast`参数时，将获得不确定的结果。
 
当配合使用`startAt`、`endAt`和`equalTo`时，如果子节点的value包含多种数据类型，将按照参数的类型排序，即只能返回这个类型的有序数据。 上面的数据如果使用 `orderBy="$value"&startAt=60` 将得到下面的结果：
```json
{
    "linhenykus" : 80,
    "pterodactyl" : 93
}
``` 
 `Object`类型数据的 value 值为 null，不会出现在结果中。
  <p style='color:red'><em>注意：如果path与value的总长度超过1000字节时，使用`orderBy="$value"`参数将搜索不到该数据。</em></p>

**orderBy="$priority"**
当使用`orderBy="$priority"`参数对数据进行排序时，数据的顺序取决于优先级以及key值。注意，优先级的值只能是数字型或字符型。
1. 1.没有优先级的节点放在最前面
2. 2.接着是数字型的优先级，按照优先级从小到大的顺序排列
3. 3.接下来是字符型的优先级，按照优先级的字典顺序排列
4. 4.当两个节点的优先级相同，就按照key值排序，数字型key在前（数值排序），其余类型的key在后（字典排序）

关于优先级更多的内容，请参见 [API文档](https://z.wilddog.com/rest/api#Priorities0)。


### Streaming

Wilddog REST端点支持 [EventSource/Server-Sent Events](http://www.w3.org/TR/eventsource/) 协议，使客户端可以持续得到指定节点下数据发生的变化。

想要开始streaming，我们需要：
 1. 将客户端请求的Accept头设置为`text/event-stream`
 2. 考虑http跳转，特别是307状态码
 3. 如果数据访问需要校验权限，需要添加`auth`参数。

当指定节点的数据发生变化时，服务器会发送事件回来。消息的结构符合`EventSource`协议：

```
event: event name
data: JSON encoded data payload
```

服务器会返回以下的事件：

|     		|		     |
|   ----    |    ----   |
|   put		|	JSON格式数据，有两个key：path和data；path指向请求URL的路径；客户端应该使用消息中包含的数据替换本地缓存中的数据   |
|   patch	|	JSON格式数据，有两个key：path和data；path指向请求URL的路径；对与数据的每一个key，客户端应该用消息中包含的key替代缓存中的key   |
| keep-alive	|	此事件的data为null，不需要任何操作   |
|  auth_revoked	|	该事件的数据是字符串，表示认证过期；认证参数失效后，该事件将被发送   |

下面是服务器能够发送的事件示例：

```json
// 设置整个缓存为 {"a": 1, "b": 2}
event: put
data: {"path": "/", "data": {"a": 1, "b": 2}}

//将新数据放到key为c的缓存中，这样整个缓存是下面的结构
// {"a": 1, "b": 2, "c": {"foo": true, "bar": false}}

event: put
data: {"path": "/c", "data": {"foo": true, "bar": false}}

// 数据中的每个key，更新（或添加）相应的key到缓存的路径/c下
// 最终的缓存为: {"a": 1, "b": 2, "c": {"foo": 3, "bar": false, "baz": 4}}
event: patch
data: {"path": "/c", "data": {"foo": 3, "baz": 4}}

```
<br>
## 5.组织数据

构造恰当的NoSQL存储结构需要事先考虑很多因素。最重要的是，必须要知道将来数据会被如何查询，如何存储数据才能使查询最方便。

### 避免层级过深

尽管可以使用JSON任意地组织数据，但不同的组织方式对读取性能的影响是很大的。Wilddog的工作方式是当你查询某个节点，Wilddog会返回这个节点下的所有子节点。所以，应该尽可能使数据扁平化，就像组织SQL关系型数据表一样。

<div class="alert"> 我们不推荐这种实践 </div>

```json
{
	// 一个非常差的充满嵌套的数据结构。请勿模仿。
    // 对"rooms"进行遍历查找来获得名字需要下载很多很多的messages。
    "rooms": {
      "one": {
        "name": "room alpha",
        "type": "private",
        "messages": {
          "m1": { "sender": "mchen", "message": "foo" },
          "m2": { ... },
          // 非常长的messages列表
        }
      }
    }
}
```

对于这种嵌套存储的设计，很难遍历所有的数据。比如列出所有的rooms这样一个很简单的操作，也会查询整个`rooms`数据节点，返回所有的rooms下的数据节点到客户端。

### 使数据扁平化

如果数据分布到不同的路径下，那么就可以根据需要查询最小化的数据量，大大提高查询性能：

```json
{
    // rooms数据节点下仅包含房间的基本信息和唯一ID。
    "rooms": {
      "one": {
        "name": "room alpha",
        "type": "private"
      },
      "two": { ... },
      "three": { ... }
    },

    //room成员可以很方便的的存取
    "members": {
      "one": {
        "mchen": true,
        "hmadi": true
      },
      "two": { ... },
      "three": { ... }

    },

    //消息数据与其他数据分离开，这样我们在查询其他数据时就不收消息数据的影响，从而提升性能。
	//消息数据可以通过room ID方便的分页和查询。
    "messages": {
      "one": {
        "m1": { "sender": "mchen", "message": "foo" },
        "m2": { ... },
        "m3": { ... }
      },
      "two": { ... },
      "three": { ... }
    }

  }
```
这样组织数据，就可以很方便的查询room列表了，只需要传输很少的字节数。message数据也可以很容易的查询。

### 使数据可扩展

很多时候需要查询一个列表的一个子集数据，尤其是当这个列表中包含多达数千条或更多记录时。当这个数据之间的关系是单向且数据比较稳定的时候，我们可以简单的把子节点数据嵌套到父节点之下：

```json
{
    "users": {
      "john": {
         "todoList": {
            "rec1": "Walk the dog",
            "rec2": "Buy milk",
            "rec3": "Win a gold medal in the Olympics"
         }
      }
    }
  }
```

但很多时候数据频发变化，或者有时候必须把数据拆分存储到不同的路径下（John可能有一个长达数千项的todo列表）。

但仅仅如此可能还是不够的。考虑一个例子，users和groups之间的双向关系。user可以属于group，group包含一个user列表。乍看之下数据可能这样组织：

```json
{
    "users": {
      "mchen": { "name": "Mary Chen" },
      "brinchen": { "name": "Byambyn Rinchen" },
      "hmadi": { "name": "Hamadi Madi" }
    },
    "groups": {
      "alpha": {
         "name": "Alpha Tango",
         "members": {
            "m1": "mchen",
            "m2": "brinchen",
            "m3": "hamadi"
         }
      },
      "bravo": { ... },
      "charlie": { ... }
    }
  }
```
看起来不错。但是当需要判断一个user属于哪些group的时候，困难就来了。我们可以在数据发生改变的时候遍历并更新所有的group，但这样做成本很高，也很慢。更糟糕的是，如果Mary没有权限查看所有的group时怎么办呢？当查询整个列表时，会得到一个没有权限访问的错误。

我们需要的是一种优雅的方式，可以列出Mary属于哪些group，只需要查询这些group就行了。数据可以这样组织：

```json
{
    "users": {
      "mchen": {
        "name": "Mary Chen",
        // 在Mary的数据下，建立他所属group的索引。
        "groups": {
           // 这里的值是什么并不重要。重要的是这个子节点的key存在。
           "alpha": true,
           "charlie": true
        }
      },
      ...
    },
    "groups": { ... }
  }
```

我们把关系数据同时存储在了Mary的记录下和group数据下，这样造成了数据的重复。如果要把Mary从一个组中删除，就需要更新两个地方。

对于双向的关系来说，这样的冗余是有必要的。这样做使我们可以很高效的查询Mary的个人信息，即使users和groups都有百万级的数据，且规则表达式禁止访问不相关的数据时。

为什么我们把id作为key，而把value设置为`true`呢？这样做是有好处的。这样使得检查一个id是否存在变得非常简单，只需要读取`/users/mchen/groups/$group_id`，看它是否为null就可以了。

<br>
## 6.了解安全
安全是一个非常重大的话题，通常也是app开发中最困难的部分之一。Wilddog使用一种声明式的规则表达式，对数据的访问权限进行配置，让这一切变得简单。

### 认证

用户ID是一个非常重要概念，不同的用户拥有不同的数据和不同的权限，比如，在一个聊天程序中，每一条消息都有它的发布者，用户可以删除自己的消息，而不能删除别人的。安全的第一步是用户认证。

Wilddog 提供了以下终端用户认证的方式：
* 集成微博，微信，QQ等社交平台的OAuth认证
* Email/密码登录，并且提供用户管理
* 匿名用户访问
* 自定义token，方便用户集成已有的用户账户系统。

### 授权

知道用户的身份只是安全的一部分，一旦你知道谁在访问数据，你需要一种方式来控制访问权限。Wilddog提供了一种声明式的表达式语言，你可以在控制面板中的“规则表达式”tab下进行编辑。这些规则表达式让你可以管理数据的访问规则。规则级联应用到其子节点。

``` json
{
  "rules": {
    "foo": {
      ".read": true，
      ".write": false
    }
  }
}
```

这个例子允许所有人访问数据节点 `foo`。

规则表达式包含一系列内置对象和函数。最重要的一个内置对象是auth，它在终端用户认证的时候生成，包含终端用户的信息和用户的唯一id：auth.uid。

auth对象是很多规则表达式的基础。

``` json
{
  "rules": {
    "users": {
      "$user_id": {
        ".write": "$user_id == auth.uid"
      }
    }
  }
}
```


这个规则保证了：只有终端用户的唯一id等于动态路径$user_id的值时，用户才能写入数据。



### 数据校验
规则表达式中还包含一个`.validate`规则，用于对数据进行校验，确保数据的格式正确。它的语法和`.read`与`.write`相同，不同的是`.validate`规则不会向下级联。

``` json
{
  "rules": {
    "foo": {
      ".validate": "newData.isString() && newData.val().length() < 100"
    }
  }
}
```

这一规则确保了在/foo/节点下，写入的数据必须是字符串类型，且必须长度小于100。

`.validate`规则可以使用的内置对象和方法与`.read`和`.write`相同。

``` json
{
  "rules": {
    "user": {
      ".validate": "auth != null && newData.val() == auth.uid"
    }
  }
}
```

这一规则强制使写入/user/下的数据必须是当前登陆用户的唯一id。

`.validate`规则并不是要彻底取消应用中的数据校验代码。为了获得更好的性能和用户体验，你仍然必须在应用代码中对数据进行校验。

### 了解更多

到此为止，你应该对Wilddog中的应用安全机制有了一个大体的了解。

规则表达式是复杂且强大的，本开发向导中只涵盖了非常小的一部分。更多关于规则表达式的细节，请参考[规则表达式文档](https://z.wilddog.com/rule/quickstart)，这里将会讲述所有的内置方法和对象。

<br>
## 7.用户认证

### 概览

大多数应用都需要识别用户的身份，针对不同的用户设置不同的数据访问权限，这一过程称为用户认证，Wilddog提供了一系列简单易用的认证方式。

当用户通过app认证时，
- １. 用户的信息将返回到客户端的回调函数中，这样你可以为指定用户定制个性化的体验。

- ２. 规则表达式中的内置对象auth将会被赋值。在用户认证之前，auth对象为null，当用户认证成功后，auth对象将会包含用户的唯一标识(auth.uid)和其它一些可能用到的数据，这样你可以基于用户身份控制数据的访问权限。

Wilddog的JavaScript，Java和Objective-C的库都提供了第三方登录的功能，但REST API有所不同，它的目标是灵活和平台无关性。你可以结合服务端生成用户的登陆的token进行自定义登录。

Wilddog App允许你使用JSON Web Tokens(JWTs)对用户的验证进行完全的控制，当你的服务器上已经有了一套用户账号管理系统，但是想满足更高级的认证需求或增强认证服务时，这种认证方法非常有用。

### 生成安全token
为了验证自定义用户的身份，我们必须在服务端为每个客户端生成一个token，野狗提供了JAVA和PHP的工具类，可以根据应用的超级密钥(控制面板超级密钥侧边栏)生成token。

java : [wilddog-token-generator-java](https://github.com/WildDogTeam/wilddog-token-generator-java)

php : [wilddog-token-generator-php](https://github.com/WildDogTeam/wilddog-token-generator-php)

java代码：

```java
Map<String, Object> authPayload = new HashMap<String, Object>();
authPayload.put("uid", "1");
authPayload.put("hasEmergencyTowel", true);

TokenGenerator tokenGenerator = new TokenGenerator("<YOUR_WILDDOG_SECRET>");
String token = tokenGenerator.createToken(authPayload);

System.out.println(token);
```

php示例代码：

```php
use Wilddog\Token\TokenGenerator;

$generator = new TokenGenerator('<YOUR_WILDDOG_SECRET>');

$token = $generator
    ->setOption('admin', true)
    ->setData(array('uid' => 'exampleID'))
    ->create();
```

注：示例代码中的 `<YOUR_WILDDOG_SECRET>` 为超级密钥。

任何传入 `createToken()` 的参数信息都会附加到 `auth` 中，以便你在配置安全和规则表达式中使用。继续上面示例，你可以在规则表达式中这样写：
```json
{
  "rules": {
    "frood": {
      ".read": "auth.hasEmergencyTowel == true"
    }
  }
}
```
在上面的规则里，只有包含 `hasEmergencyTowel = true` 参数信息的 token 才有在 frood 节点下读取数据的权限。

token的默认过期时间是颁发后的24小时，到达过期时间客户端会自动取消授权，你可以在控制面板的终端用户认证中修改session的过期时间，或者在创建token的时候指定过期时间。

为了处理token过期的问题，当认证状态变化后，各个平台(web,iOS,Android)都提供了相应的方法(onAuth,observeAuthEventWithBlock)进行回调。

### 验证请求

一旦JWT数据生成，我们就可以通过附加auth参数的请求访问受保护的数据，这个auth可以是超级密钥或自定义的token。
```
curl 'https://<appId>.wilddogio.com/rest/saving-data/auth-example.json?auth=CREDENTIAL'
```
### 认证服务器

你可以用以下方式进行认证：
1. 使用野狗的超级密钥：所有的验证方法都可以接收一个超级密钥取代JWT token，这将赋予对整个野狗数据库的读写权限，在超级密钥被废弃之前，这个认证会一直生效。
2.  使用JWT Token并将admin声明为true：这种方法也是授予了对整个野狗数据库的读写权限，token会正常的过期。
3.  使用JWT Token访问指定的数据片段：这种方法比较复杂，但却是最安全的对数据访问控制的方法。

### 使用lib库生成token

我们后续将会提供其他各种语言平台上的token生成工具。用户也可以自行生成token，只需符合格式约定。token采用标准的jwt格式，payload部分中，必须包含的字段如下：

| 字段 | 描述 |
| --- | --- |
| v | token的版本，默认是数字0  |
| iat | token的颁发时间，Unix时间秒数 |
| d | 认证数据。token的payload，必须包含uid字段，对应规则表达式中的`auth`变量|

下面是可选参数：

| 字段 | 描述 |
| --- | --- |
| nbf | token在之前（缩写"not before"）时间不会生效|
| exp | token过期的时间戳，以秒为单位 |
| admin | 如果设置为true，将获得完全的读写权限|
| debug | 如果设置为true，将在安全和规则表达式失败时提供详细的错误信息|

一个示例的token payload：
``` json
{
    "v" : 0,
    "iat" : "1437520447",
    "d" : {
        "uid" : "sampleId"
    },
    "admin" : true,
    "exp" :  1437845927
}
```
<br>
## 8.应用管理

### 安全规则设置

使用REST API，你可以通过发送PUT请求到`/.settings/rules.json`路径为你的应用设置规则表达式。请求时你需要使用野狗的超级密钥，例如为了将所有读权限设置为true，可以发送如下的请求：
```
curl -X PUT -d '{"rules":{".read":true}}' 'https://<appId>.wilddogio.com/.settings/rules.json?auth=WILDDOG_SECRET'
```
通过REST API编写应用的安全规则表达式可以作为你应用部署流程中的一部分。

注：通过REST API修改rules将会覆盖已经存在的规则表达式。

### 获取安全规则

类似的，我们可以通过发送GET请求到`/.settings/rules.json`路径来获取规则表达式的设置内容：
```
curl  'https://<appId>.wilddogio.com/.settings/rules.json?auth=WILDDOG_SECRET'
```
返回的数据将包含app设置的规则表达式的全部内容。




















