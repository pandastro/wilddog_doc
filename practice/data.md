/*
Title : 组织数据
Sort : 1
Tmpl : page
*/

构造恰当的NoSQL存储结构需要事先考虑很多因素。最重要的是，必须要知道将来数据会被如何查询，如何存储数据才能使查询最方便。

<br>

## 避免层级过深

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

#### 使数据扁平化

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

<br>

## 使数据可扩展
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

但很多时候数据频发变化，或者有时候必须把数据拆分存储到不同的路径下（John可能有一个长达数千项的todo列表）。通常可以通过查询一个列表的子集的方式来解决。

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

如果我们要判断Mary是否属于 alpha group，以 javascript 代码为例：
```js
// 判断Mary是否属于alpha group
var ref = new Wilddog("https://docs-examples.wilddogio.com/web/org/users/mchen/groups/alpha");
ref.once('value', function(snap) {
  var result = snap.val() === null? 'is not' : 'is';
  console.log('Mary ' + result + ' a member of alpha group');
});
```
----