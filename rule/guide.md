/*
Title : 开发向导
Sort : 1
Tmpl : page
*/

## 1. 了解安全

安全是一个非常重大的话题，通常也是app开发中最困难的部分之一。Wilddog使用一种声明式的规则表达式，对数据的访问权限进行配置，让这一切变得简单。

### 认证

用户ID是一个非常重要概念，不同的用户拥有不同的数据和不同的权限，比如，在一个聊天程序中，每一条消息都有它的发布者，用户可以删除自己的消息，而不能删除别人的。安全的第一步是用户认证。

Wilddog 提供了以下终端用户认证的方式：
* 集成微博，微信，QQ等社交平台的OAuth认证
* Email/密码登录，并且提供用户管理
* 匿名用户访问
* 自定义token，方便用户集成已有的用户账户系统。
* 
<br>
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

## 2. 数据安全

### 概述

Wilddog提供了一个强大的，基于表达式的，类似javascript的规则表达式语言。规则表达式符合JSON规范，可用于访问权限控制和数据完整性校验。

规则是声明式的，即独立于主产品逻辑之外。这样是有好处的：安全性不完全依赖于客户端，无需中间服务器即可保护数据安全，真正解决无后端应用中的安全问题。

### 规则种类

规则表达式共有三个类型，如下表：

| 规则类型 | 描述 |
| --- | --- |
| .read | 定义了数据是否可以被用户读取 | 
| .write | 定义了数据是否可以被用户写入 |
| .validate | 定义了什么样的数据是正确的格式，是否有某些子属性，数据类型等 | 

规则被存储为JSON格式，如下：

```json
{
  "rules": {
    "foo": {
      // /foo节点可被任何人读取。
      ".read": true,
      
      // /foo节点可被任何人写入。
      ".write": true,
      
      // 写入/foo 的数据必须是字符串类型且长度小于100。
      ".validate": "newData.isString() && newData.val().length() < 100"
    }
  }
}
```
如果没有为指定的节点赋予读和写的规则表达式（如上边代码中数据的根节点），那么它们将默认为false。

### 内置变量

在规则表达式中，可以使用一些内置变量。下面的实例中将会涵盖到大部分内置变量，这里先列出一个简要描述：

内置对象     | 描述
-------- | ---
now| 云端的时间戳，以毫秒为单位。
root|`RuleDataSnapshot`类型的对象，代表操作之前，数据根节点`/`的数据的引用。
newData| `RuleDataSnapshot`类型的对象，代表假设数据操作成功，之后此节点下的数据，也就是此节点下的旧数据和本次操作写入的新数据合并之后的数据。
data| `RuleDataSnapshot`类型的对象，代表此节点被操作前的原始数据。
$variables| 节点的名称变量，代表动态路径的通配符。 
auth| `Auth`类型对象，代表已经登录用户对象。

### data和newData

内置对象data指的是写操作发生之前，当前节点的已有数据。newData则指的是，假定写操作会成功，那么写操作成功之后当前节点的数据。newData代表的是旧数据和即将写入的数据合并之后的数据。

为了演示它们的用法，考虑一个这样的场景：我们需要一个规则，在指定的路径下，当前节点数据不存在的时候可以写入，也可以删除已存在的数据，但不能对已有数据进行修改：

```json
// data不存（没有旧数据）在或者newData不存在（删除数据）的情况下，可以写入。
// 也就是说可以是全新写入，也可以是删除，但是不能修改已有数据。
".write": "!data.exists() || !newData.exists()"
```

### 引用其它路径的数据

通过使用内置变量`root`，`data`和`newData`，你可以访问到任何数据节点。

参考下面的例子，只有当`allow_writes`节点的值为`true`，且父节点没有设置一个`readOnly`标志，且新写入的数据中存在名为`foo`的子节点时，数据才被允许写入。

```json
{
  ".write": "root.child('allow_writes').val() == true &&
            !data.parent().child('readOnly').exists() &&
            newData.child('foo').exists()"
}
```

### 规则的级联

这是一个非常重要的特性！规则表达式遵循一个自上向下延展的原则。如果一个数据节点上的`.write`或`.read`规则赋予了读或写的权限，那么它的所有的子节点也都将拥有读或写的权限。子节点上的规则不能收回父节点或祖先节点已经赋予的读或写的权限。参考下面的例子：

``` json
  "rules": {
     "foo": {
        // 允许/foo/节点下的数据被读取
        ".read": "data.child('baz').val() == true",
        "bar": {
          // 当父节点的表达式授予了读权限时，这一规则设置false无效。
          ".read": false
        }
     }
  }
}
```

在这个规则的作用下，只要当`/foo/`节点包含一个名为`baz`且值为`true`的子节点时，`/bar`就能被读取。`/bar`下的规则`".read":false`不起任何作用。因为.read的默认值本来就是`false`，而且当`/foo`的`.read`规则运行结果为`true`时，子节点`bar`也不能收回已赋予的权限。

### 规则不是过滤器

规则的执行是原子性的。在一次操作中，只要任何一个数据节点没有访问权限，那么整个操作将会失败。参考这个例子：

```json
{
  "rules": {
    "records": {
      "rec1": {
        ".read": true
      },
      "rec2": {
        ".read": false
      }
    }
  }
}
```

如果不理解规则执行的原子性，很可能会误以为读取路径`/records`将会返回`rec1`，不返回`rec2`。然而实际的结果是，整个读取操作返回一个错误。

由于读操作是原子性的，`rec2`是不可读的，因此将会返回一个没有操作权限的错误。

### 使用$Variables

规则路径中以`$`开头的变量是通配的，它的值可以在规则表达式中被使用：

```json
{
  "rules": {
    "rooms": {
      // 下面的规则适用于/rooms/下的所有子节点。/rooms/下的每一个room的id
      // 都被存储在变量 $room_id中，以备在表达式中使用。
      "$room_id": {
        "topic": {
          // 如果room的id中包含public，那么它的topic就可以被修改。
          ".write": "$room_id.contains('public')"
        }
      }
    }
  }
}
```

也可以如下面这样使用，限定在`/widgets/`节点下，任何没有明确列出的子节点都不能被写入：

```json
{
  "rules": {
    "widget": {
      // 一个widget可以有title和color属性。
      "title": { ".validate": true },
      "color": { ".validate": true },
      
      // 但是其他的任何子节点都不允许有
      "$other": { ".validate": false }
    }
  }
}
```

需要注意的是，规则路径中的动态变量总是字符串类型的。如果需要把`$variables`和一个数值进行比较，将总是返回false。正确的写法是将数值转换为字符串（如：$key == newData.val() + ''）。


### 数据校验

通过`.validate`规则，可以保障数据的组织形式和格式。`.validate`规则在`.write`规则执行成功之后执行。

`.validate`规则不会向下级联。如果在一次操作中，任何一个子节点的`.validate`规则失败，整个写操作都将失败。当数据被删除（也就是值为null）时，`.validate`表达式被忽略。

参考下面的规则：

```json
{
  "rules": {
    // write is allowed for all paths
    ".write": true,
    "widget": {
      // a valid widget must have attributes "color" and "size
      // allows deleting widgets (since .validate is not applied to delete rules)
      ".validate": "newData.hasChildren(['color', 'size'])",
      "size": {
        // the value of "size" must be a number between 0 and 99
        ".validate": "newData.isNumber() &&
                      newData.val() >= 0 &&
                      newData.val() <= 99"
      },

      "color": {
        // the value of "color" must exist as a key in our mythical
        // /valid_colors/ index
        ".validate": "root.child('valid_colors/' + newData.val()).exists()"
      }
    }
  }
}

```

下面用javascript SDK作为示例，展示了各种写操作的结果。
```js
var ref = new Wilddog(URL + "/widget");

// 写入失败: 没有color和size子节点
ref.set('foo');

// 写入失败: 没有color子节点
ref.set({size: 22});

// 写入失败：size子节点不是数值
ref.set({ size: 'foo', color: 'red' });

// 写入成功：
ref.set({ size: 21, color: 'blue'});

// 如果节点存在且有color子节点, 操作将会成功, 否则失败，因为newData.hasChildren(['color', 'size'])将会失败。
ref.child('size').set(99);
```

### 匿名聊天的例子

下面是一个匿名聊天的App的规则示例：

```json
{
  "rules": {
    // 任何节点都允许写数据
    ".write": true,
    
    "widget": {
      // widget必须包含 "color"和"size"子节点
      // 删除widget是可以的，因为.validate不会作用于删除
      ".validate": "newData.hasChildren(['color', 'size'])",
      "size": {
        // size必须是数值，且0到99之间。
        ".validate": "newData.isNumber() &&
                      newData.val() >= 0 &&
                      newData.val() <= 99"
      },
      "color": {
        // color的值必须在/valid_colors/之下存在。
        ".validate": "root.child('valid_colors/' + newData.val()).exists()"
      }
    }
  }
}

```

<br>

## 3. 基于终端用户认证的安全

### 概述

本小节介绍如何将规则表达式与终端用户认证结合起来，实现基于用户的数据访问权限控制。

### 内置对象auth

在进行用户终端认证之前，内置对象auth为null。一旦进行了终端用户认证，那么auth对象将会存在，并且包含以下属性：

属性     | 描述
-------- | ---
provider | 终端用户认证所使用的方式。可为password, anonymous, weibo, weixin等。
uid   | 用户的唯一id，在多重不同的认证方式之间仍然可以确保唯一

下面是一个简单的示例：

```json
{
  "rules": {
    "users": {
      "$user_id": {
        // 只有用户自己才有写权限
        ".write": "$user_id == auth.uid"
      }
    }
  }
}
```

<br>
## 4. 数据索引

Wilddog允许你按照任意子节点的顺序对数据进行查询。如果你预先确定了需要按顺序查询的节点，可以在规则表达式中通过`.indexOn`规则为节点数据建立索引，以此来提高查询效率。


### 定义数据索引

Wilddog提供了强大的工具来排列和查询数据，这样你可以按照任意子节点的顺序对数据进行查询。随着你的应用数据量的增加，查询效率会逐渐降低。但是，如果在Wilddog设置了你想要查询的节点，Wilddog就可以为节点数据建立索引，以此来提高查询效率。

节点的名称key和优先级priority默认建立索引，不需要额外设置。

### orderByChild索引

我们通过一个例子来讲解如何通过`.indexOn`来建立`orderByChild`索引。下面是关于恐龙的一些数据信息：
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
假设在应用中，我们需要经常对恐龙信息按照名称、高度、长度进行排序，但是不会按照体重排序，这样我们就可以通过Wilddog提供的`.indexOn`规则，为名称、高度、长度这些节点建立索引，来提高查询效率。恐龙的名称即为节点的key值，已经默认建立了索引，所以不需要额外设置。我们可以使用如下的设置为高度、长度节点建立索引：
```
{
  "rules": {
    "dinosaurs": {
      ".indexOn": ["height", "length"]
    }
  }
}
```
跟其他的规则一样，你可以在任意节点下设置`.indexOn`规则。
上面的例子中，我们将`.indexOn`规则放在`/dinosaurs`节点下，这是因为所有的恐龙数据都是存放在`/dinosaurs`节点下。


### orderByValue索引

下面的例子中，我们将讲解如何通过`.indexOn`来建立`orderByValue`索引。假如我们现在要为恐龙建立一个排行榜，数据如下：
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
由于是按照节点的value值进行的排序，我们可以在`/scores`节点下增加`".indexOn": ".value"`规则来优化查询，例如：
```
{
  "rules": {
    "scores": {
      ".indexOn": ".value"
    }
  }
}
```

<br>
## 5. 自定义token

### 使用自定义token

当你选择自定义用户数据库进行终端用户认证时，将会用到自定义token进行自定义身份认证。

### 工具生成token

你可以自行生成Wilddog终端用户所认证所使用的token。我们提供了token生成工具：

java : [wilddog-token-generator-java](https://github.com/WildDogTeam/wilddog-token-generator-java)

php : [wilddog-token-generator-php](https://github.com/WildDogTeam/wilddog-token-generator-php)

java版生成工具依赖：
```pom
<dependency>
    <groupId>com.wilddog</groupId>
    <artifactId>wilddog-token-generator</artifactId>
    <version>1.0.0</version>
</dependency>
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
java代码：

```java
Map<String, Object> authPayload = new HashMap<String, Object>();
authPayload.put("uid", "1");
authPayload.put("hasEmergencyTowel", true);

TokenGenerator tokenGenerator = new TokenGenerator("<YOUR_WILDDOG_SECRET>");
String token = tokenGenerator.createToken(authPayload);

System.out.println(token);
```
注：示例代码中的 ```<YOUR_WILDDOG_SECRET>``` 为超级密钥。超级密钥可以帮你实现你直接与野狗服务器通信，拥有对数据的最高的读写权限。你可以在每个应用对应的控制面板中得到、增加或废弃超级密钥。

任何传入 ```createToken()``` 的参数信息都会附加到 ```auth``` 中，以便你在配置安全和规则表达式中使用。继续上面示例，你可以在规则表达式中这样写：
```json
{
  "rules": {
    "frood": {
      ".read": "auth.hasEmergencyTowel == true"
    }
  }
}
```
在上面的规则里，只有包含 ```hasEmergencyTowel = true``` 参数信息的 token 才有在 frood 节点下读取数据的权限。

你甚至可以自定义token的生效时间、过期时间等，查阅 [wilddog-token-generator-java_README.md](https://github.com/WildDogTeam/wilddog-token-generator-java/blob/master/README.md) 以获取更多的信息。

### 自行生成token

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
    "iat" : 1437520447,
    "d" : {
        "uid" : "sampleId"
    },
    "admin" : true,
    "exp" :  1437845927
}
```

使用SHA-256 HMAC签名，生成标准的jwt即可。




