/*
Title: 读写数据
Sort: 1
Tmpl : page
*/

## 写入数据

在本文档中，我们将介绍向数据库写入数据的方法:`setValue()`, `updateChildren()`, `push()`, `runTransaction( )`。

### 保存数据的方法
接口 | 描述
---- | ----
`setValue()` | 覆盖指定路径的所有数据，如messages/users/<username>
`updateChildren()` | 对子节点进行合并操作。不存在的子节点将会被新增，存在的子节点将会被替换。
`push()` | 在当前节点下生成一个子节点，并返回子节点的引用。子节点的key利用服务端的当前时间生成，可作为排序使用。
`runTransaction( )` | 一个复杂数据被并发更新导致数据错误，使用事务防止数据被并发更新。

### 使用setValue()写入数据
wilddog通常使用`setValue()`方法来写入数据，该方法用来覆盖指定路径上的所有数据。为了更好地理解该方法，我们建立一个博客app来说明。app的数据保存在下面引用对应的路径中：
```Java
Wilddog ref = new Wilddog("https://docs-examples.wilddogio.com/android/saving-data/wildblog");
```
我们添加一些用户，为每个用户保存唯一的用户名，同时保存全名和出生日期。由于每个用户的用户名都是独一无二的，所以最好使用setValue()方法，而不是push()方法，因为我们已经有了独一无二的用户名作为key值，不需要在添加的时候重新生成唯一标识。

首先，我们编写User类代码，将User对象以用户名作为key值添加到Map中。然后，为用户数据所在路径创建引用，调用setValue()方法将Map中的每个用户添加到数据库中。
```Java
public class User {
    private int birthYear;
    private String fullName;

    public User() {}

    public User(String fullName, int birthYear) {
        this.fullName = fullName;
        this.birthYear = birthYear;
    }

    public long getBirthYear() {
        return birthYear;
    }

    public String getFullName() {
        return fullName;
    }
}

User alanisawesome = new User("Alan Turing", 1912);
User gracehop = new User("Grace Hopper", 1906);

Map<String, User> users = new HashMap<String, User>();
users.put("alanisawesome", alanisawesome);
users.put("gracehop", gracehop);

Wilddog usersRef = ref.child("users");

usersRef.setValue(users);
```
我们可以向setValue()方法传入自定义的Java对象作为参数，但需要满足如下条件：
1. 对象所属的类中存在默认的构造方法; 
2. 类中所有需要写入数据库的属性都有getter方法。

我们使用Map将数据保存到数据库中，因为Map中的元素会自动映射成为JSON对象，并保存到指定路径。现在，我们访问 `https://docs-examples.wilddogio.com/android/saving-data/wildblog/users/alanisawesome/fullName`，将会看到返回值"Alan Turing"。我们也可以直接将数据保存到数据库的指定路径：
```Java
//使用父节点引用的child()方法，获得指向子数据节点的引用。
usersRef.child("alanisawesome").child("fullName").setValue("Alan Turing");
usersRef.child("alanisawesome").child("birthYear").setValue(1912);

// 也可以在child()方法的参数中使用/分隔多个子节点路径。
usersRef.child("gracehop/name").setValue("Grace Hopper");
usersRef.child("gracehop/birthYear").setValue(1906);
```

上面介绍的两种保存数据的方式，一种是使用Map将所有数据一次性保存到数据库，另一种是将数据分别保存到数据库的指定路径，最终的效果都是一样的：
```JSON
{
  "users": {
    "alanisawesome": {
      "birthYear": "1912",
      "fullName": "Alan Turing"
    },
    "gracehop": {
      "birthYear": "1906",
      "fullName": "Grace Hopper"
    }
  }
}
```
使用`setValue()`方法将会覆盖指定路径的所有数据，包括子节点的数据。
我们向`setValue()`方法传递的参数类型需要与JSON中能用的类型对应，如`String`， `Long`， `Double`， `Boolean`， `Map<String, Object>` 和 `List<Object>`。使用这些数据类型，我们可以构造任意复杂的数据结构，例如Map中嵌套另一个Map。我们可以不使用User对象，而使用Map来实现与上面相同的功能：
```Java
Wilddog usersRef = ref.child("users");

Map<String, String> alanisawesomeMap = new HashMap<String, String>();
alanisawesomeMap.put("birthYear", "1912");
alanisawesomeMap.put("fullName", "Alan Turing");

Map<String, String> gracehopMap = new HashMap<String, String>();
gracehopMap.put("birthYear", "1906");
gracehopMap.put("fullName", "Grace Hopper");

Map<String, Map<String, String>> users = new HashMap<String, Map<String, String>>();
users.put("alanisawesome", alanisawesomeMap);
users.put("gracehop", gracehopMap);

usersRef.setValue(users);
```
注意：如果向`setValue()`方法传递null作为参数，将会删除指定路径的数据。

### 更新已保存的数据
如果想不覆盖路径下的所有数据，只更新部分数据，则可以使用`updateChildren()`方法，例如：
```Java
Wilddog hopperRef = usersRef.child("gracehop");
Map<String, Object> nickname = new HashMap<String, Object>();
nickname.put("nickname", "Amazing Grace");
hopperRef.updateChildren(nickname);
```
以上的代码将会为用户新增nickname属性。如果我们此处使用`setValue()`方法，则用户的fullName和birthYear属性将会被删除，只保留nickname属性。
*注意*: 将null作为参数传给`updateChildren()`，会将指定位置的数据删除。

### 增加回调方法
如果你想知道数据是否已经成功保存，可以增加一个回调方法。`setValue()`和`updateChildren()`方法都支持回调方法，当数据保存失败时，将会返回错误信息：
```Java
ref.setValue("I'm writing data", new Wilddog.CompletionListener() {
    @Override
    public void onComplete(WilddogError wilddogError, Wilddog wilddog) {
        if (wilddogError != null) {
            System.out.println("Data could not be saved. " + wilddogError.getMessage());
        } else {
            System.out.println("Data saved successfully.");
        }
    }
});
```

### 向列表中添加数据
当需要向数据库列表中添加数据时，我们需要注意应用通常都是多用户的，应根据实际情况调整列表结构。扩展我们上面的实例，假如我们要向博客app中新增posts数据。可能你首先想到的是使用`setValue()`方法保存数据，使用自增的序列来作为元素的key值，结构如下：
```Java
// 不推荐用这种方法，建议使用push()方法!
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
如果用户要在/posts/2节点处增加一个posts，当只有一个用户操作时是没有问题的，但是实际情况是可能会有多个用户同时添加，当有两个用户同时向/posts/2节点添加数据时，其中一个posts会被另一个覆盖。

为了解决这个问题，wilddog提供了`push()`方法，使用该方法每次新加元素时，都会为元素生成一个唯一标识，通过这种方式，多个客户端可以同时向同一个位置添加元素。`push()`生成的唯一标识是基于时间戳计算得来的，所以列表元素是按照时间顺序排列的。
我们可以通过下面的方式来向博客app写入posts数据：
```Java
Wilddog postRef = ref.child("posts");

Map<String, String> post1 = new HashMap<String, String>();
post1.put("author", "gracehop");
post1.put("title", "Announcing COBOL, a New Programming Language");
postRef.push().setValue(post1);

Map<String, String> post2 = new HashMap<String, String>();
post2.put("author", "alanisawesome");
post2.put("title", "The Turing Machine");
postRef.push().setValue(post2);
```
由于使用了`push()`方法为每个post数据生成了基于时间戳的唯一标识，即使多个用户同时添加post也不会产生冲突。Wilddog数据库中的数据结构如下：
```Java
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
#### 获取`push()`方法生成的唯一标识

调用`push()`方法将会返回一个指向新路径的引用，我们可以通过这个引用来获取唯一标识，或者在新路径中添加数据。下面的代码可以实现上面实例的功能，同时我们可以获取`push()`方法生成的唯一标识：

```Java
// 使用push()方法生成一个指向新路径的引用，然后使用写入数据
Wilddog postRef = ref.child("posts");
Wilddog newPostRef = postRef.push();

// 向新路径写入数据
Map<String, String> post1 = new HashMap<String, String>();
post1.put("author", "gracehop");
post1.put("title", "Announcing COBOL, a New Programming Language");
newPostRef.setValue(post1);

// 获取由push()方法生成的唯一标识
String postId = newPostRef.getKey();
```
通过getKey()方法就可以获取生成的唯一标识。

<hr>

## 事件类型
wilddog支持四种事件：`Value`， `Child Added`， `Child Changed`， `Child Removed`。分别对应四个callback方法`onDataChange()`，`onChildAdded()`，`onChildRemoved()`，`childChanged()`。

**value**
`value` 事件用来读取当前节点的数据快照， `value` 事件在初次获取到数据时被触发一次，此后每当数据发生变化都会被触发。回调方法`onDataChange()`被执行时候，当前节点下所有数据的快照会被作为参数传入。


**child_added**
与`value`事件返回指定数据节点下的所有数据不同，`onChildAdded`事件会为每一个现存的子节点数据触发一次，此后每当有子节点数据被增加时被触发一次。回调方法传入的也是DataSnapshot对象，包含的是新增子节点的数据。

如果我们想获取新增的博客数据，可以使用`addChildEventListener()`方法，绑定一个`ChildEventListener`，并重写它的`onChildAdded()`方法：

```java
// 获得一个Wilddog引用，指向post数据。
Wilddog ref = new Wilddog("https://docs-examples.wilddogio.com/android/saving-data/wildblog/posts");
ref.addChildEventListener(new ChildEventListener() {
    // 获得增加到posts节点下的数据
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChildKey) {
        Map<String, Object> newPost = (Map<String, Object>) snapshot.getValue();
        System.out.println("Author: " + newPost.get("author"));
        System.out.println("Title: " + newPost.get("title"));
    }
    
    //... ChildEventListener类还有onChildChanged(), onChildRemoved()方法.
    // 下面的例子中我们会重写这些方法。
});
```
在这个例子中，snapshot包含了一个post的数据。我们调用它的getValue()方法，并将返回的Object转换为一个Map，然后从这个map中可以得到这个post的author和title字段的数据。在大多数情况下，我们都需要先做一个判断：返回不是null，而且是一个map。

**child_changed**
当子节点数据发生了改变时，`onChildChanged`事件会被触发，事件回调方法中传入的参数是子节点改变后的数据快照。
我们可以使用`onChildChanged`事件来读取被修改的博客数据：

```java

@Override
public void onChildChanged(DataSnapshot snapshot, String previousChildKey) {
    String title = (String) snapshot.child("title").getValue();
    System.out.println("The updated post title is " + title);
}
```

**child_removed**
当直接子节点被删除时，`onChildRemoved` 事件被触发。事件回调方法中传入的参数是被删除的直接子节点的数据快照。

在博客的例子中，我们可以使用`onChildRemoved`事件，在有博客数据被删除时，在控制台中打印出来：

```java
@Override
public void onChildRemoved(DataSnapshot snapshot) {
    String title = (String) snapshot.child("title").getValue();
    System.out.println("The blog post titled " + title + " has been deleted");
}
```
<hr>

## 读取数据

Wilddog通过绑定一个异步的EventListener来读取数据。listener将会在数据初次被获取到的时候执行一次，此后每次数据发生变化都会被执行。让我们继续以博客app的例子，看看如何从云端读取数据。我们可以使用 `addValueEventListener()` 监听一个数据节点的变化。

```Java


    Wilddog ref = new Wilddog("https://docs-examples.wilddogio.com/android/saving-data/wildblog/posts");
    // 设置一个Listener，用来读取数据
	ref.addValueEventListener(new ValueEventListener(){
		public void onDataChange(DataSnapshot snapshot) {
			System.out.println(snapshot.getValue());
		}

		public void onCancelled(WilddogError error) {
			if(error != null){
				System.out.println(error.getCode());
			}
		}
	}
```

Listener方法接收到一个Snapshot对象作为参数。Snapshot对象是数据的快照。调用`Snapshot.getValue()`方法返回一个Java对象，类型可能为`Boolean`， `String`， `Number`， `Map<String, Object>` 或 `null`。如果没有数据存在，将返回`null`。

上面的例子中，我们使用`addValueEventListener()`方法读取了数据。`value`只是多种事件之一。



#### 取消事件回调
调用Wilddog引用的`removeEventListener()`方法，可以取消事件的绑定。

```java
ref.removeEventListener(originalListener);
```

如果一个listener被多次add到同一个数据地址上，那么对于每一次事件都会多次调用回调方法。要彻底取消也就需要多次调用`removeEventListener()`方法。

对一个父节点调用`removeEventListener()`，不会取消注册在子节点上的监听器。

#### 一次性读取数据
有时候需要回调方法只被执行一次就被取消掉。可以使用`addListenerForSingleValueEvent()`方法实现这一功能。

``` java
ref.addListenerForSingleValueEvent(new ValueEventListener() {
    @Override
    public void onDataChange(DataSnapshot snapshot) {
        // 执行相应的处理。 只会被触发一次。
    }
    @Override
    public void onCancelled(WilddogError error) {
    }
});
```

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

我们先编写一个代表恐龙的实体类：

```java
public class DinosaurFacts {
  long height;
  double length;
  long weight;
  public DinosaurFacts() {
    
  }
  public long getHeight() {
    return height;
  }
  public double getLength() {
    return length;
  }
  public long getWeight() {
    return weight;
  }
}
```

下面的示例中，我们将使用这个类。数据的排序有4种：按子节点，节点名，值，和优先级。

**按照指定的子节点排序**
通过将子节点的路径名作为参数调用`orderByChild()`方法，可以实现按指定子节点排序。例如，要按照height进行排序，可以：

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByChild("height");
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        DinosaurFacts facts = snapshot.getValue(DinosaurFacts.class);
        System.out.println(snapshot.getKey() + " was " + facts.getHeight() + " meters tall");
    }
    // ....
});
```

不包含指定子节点的数据节点，将会按照该子节点为null进行排序，会排在最前边。

每个查询都只能有一个排序。在一个查询中多次调用`orderByChild()`会抛出错误。


**按照数据节点名称排序**
使用`orderByKey()`方法，可以实现按照数据节点的名称进行排序。下面的例子按照alpha字母顺序读取所有的恐龙数据：

``` java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByKey();
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        System.out.println(snapshot.getKey());
    }
    // ....
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

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = scoresRef.orderByValue();
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChildKey) {
      System.out.println("The " + snapshot.getKey() + " dinosaur's score is " + snapshot.getValue());
    }
    // ....
});
```

**按照优先级排序**
关于优先级，请参考API文档中的相关部分。

#### 复杂查询
我们已经了解到数据排序的方法。接下来是limit类的和range类的查询，通过它们，可以构建出更为复杂的查询：

**limit查询**
`limitToFirst()`和`limitToLast()`两个方法用于设置最大多少个子节点数据会被同步。如果我们设置为100，那么初次获取到数据时，就只会最多触发100次事件回调方法。如果数据库中少于100条消息数据，`child_added`事件将会为每一条消息数据被触发一次。如果消息数量多于100条，那么只有其中的100条会被触发`child_added`事件。如果我们使用了`limitToFirst()`方法，那么被触发的100条将会是排序最前边的100条。如果使用了`limitToLast()`方法，那么被触发的100条将是排序最后的100条。

继续恐龙的例子，我们可以获得体重最大的两种恐龙：

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByChild("weight").limitToLast(2);
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        System.out.println(snapshot.getKey());
    }
    // ....
});
```

我们为`child_added`事件绑定的回调方法只会被执行2次。

同理，我们可以使用`limitToFirst()`方法查询最矮的两种恐龙：

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByChild("height").limitToFirst(2);
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        System.out.println(snapshot.getKey());
    }
    // ....
});
```

我们也可以组合`orderByValue()`方法来使用limit类的查询。如果要构造出恐龙运动会得分的前3名，我们可以构造这样一个查询：

```java
Wilddog scoresRef = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = scoresRef.orderByValue().limitToLast(3);
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChildKey) {
        System.out.println("The " + snapshot.getKey() + " dinosaur's score is " + snapshot.getValue());
    }
    // ....
});
```

**range查询**
使用`startAt()`，`endAt()`，和`equalTo()`方法，可以为我们的查询指定任意的起止范围。例如，如果要查询所有3米高以上的恐龙，可以组合`orderByChild()`和`startAt()`查询：

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByChild("height").startAt(3);
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        System.out.println(snapshot.getKey());
    }
    // ....
});
```

我们可以使用`endAt()`来查询按照字母排序，所有名字排在Pterodactyl之前的恐龙：

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByKey().endAt("pterodactyl");
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        System.out.println(snapshot.getKey());
    }
    // ....
});
```

注意，`startAt()`和`endAt()`都是包含边界值的，也就是说“pterodactyl”符合上边的查询条件。

我们可以同时使用`startAt()`和`endAt()`来限定一个范围。下面的例子查询出所有名字以字母“b”开头的恐龙：

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByKey().startAt("b").endAt("b\uFFFF");
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        System.out.println(snapshot.getKey());
    }
    // ....
});
```

这个例子中使用的"\uFFFF”符号是unicode中的最大字符。因为查询过程中对于字符而言,是通过unicode编码后进行排序，所以这个查询能匹配所有以b开头的值。同样这种限定范围对于汉字字符串也同样适用。

使用`equalTo()`方法，可以进行精准的查询。例如，查询所有的25米高的恐龙：

```java
Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
Query queryRef = ref.orderByChild("height").equalTo(25);
queryRef.addChildEventListener(new ChildEventListener() {
    @Override
    public void onChildAdded(DataSnapshot snapshot, String previousChild) {
        System.out.println(snapshot.getKey());
    }
    // ....
});
```

同样的，`startAt()`，`endAt()`方法也可以和`orderByValue()`方法组合进行range查询。


**总结**
组合这些方法，我们可以构造出各种复杂的查询。例如，要找出高度小于Stegosaurus但最接近的恐龙的名字：

```java
final Wilddog ref = new Wilddog("https://dinosaur-facts.wilddogio.com/dinosaurs");
ref.child("stegosaurus").child("height").addListenerForSingleValueEvent(new ValueEventListener() {
    @Override
    public void onDataChange(DataSnapshot stegosaurusHeightSnapshot) {
        Long favoriteDinoHeight = stegosaurusHeightSnapshot.getValue(Long.class);
        Query queryRef = ref.orderByChild("height").endAt(favoriteDinoHeight).limitToLast(2);
        queryRef.addListenerForSingleValueEvent(new ValueEventListener() {
            @Override
            public void onDataChange(DataSnapshot querySnapshot) {
                if (querySnapshot.getChildrenCount() == 2) {
                    DataSnapshot dinosaur = querySnapshot.getChildren().iterator().next();
                    System.out.println("The dinosaur just shorter than the stegasaurus is " + dinosaur.getKey());
                } else {
                    System.out.println("The stegosaurus is the shortest dino");
                }
            }
            @Override
            public void onCancelled(WilddogError error) {
            }
        });
    }
    @Override
    public void onCancelled(WilddogError error) {
    }
});
```



## 数据规则
本小节介绍在使用各种排序方式时，数据究竟是如何排序的。

**orderByChild**

<!--
当使用`orderByChild()`时，包含指定字段的数据将会按照以下规则排序：

1, 指定字段的值为`null`的子节点数据排在最前边。

2, 接下来是指定字段的值为`false`的子节点数据。如果存在多个子节点该字段的值为`false`，那么这些子节点根据节点名按字典序排列。

3,  接下来是指定字段的值为`true`的子节点数据。如果存在多个子节点该字段的值为`true`，那么这些子节点根据节点名按字典序排列。

4, 接下来是指定字段的值为数值类型的子节点数据，按照升序排列。如果存在多个子节点该指定字段的值相同，那么这些子节点数据按照节点名排序。

5, 接下来是字符串类型的值，按照字典序升序排列。如果存在多个子节点该指定字段的值相同，那么这些子节点数据按照节点名排序。

6, 最后是对象类型的值，按照节点名升序排列。
-->

当使用`orderByChild(key)`时，按照子节点的公有属性key的value进行排序。仅当value为单一的数据类型时，排序有意义。如果key属性有多种数据类型时，排序不固定，此时不建议使用`orderByChild(key)`获取全量数据，例如，
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
霸王龙的分数是`string`类型，雷龙的分数是`boolean`类型，而其他恐龙的分数是`numberic`类型，此时使用`orderByChild(key)`获得全量数据时，是一个看似固定的排序结果；但是配合使用`limitToFirst()`时，将获得不确定的结果。`Object`类型数据的 value 值为 null，不会出现在结果中。
当配合使用`startAt()`、`endAt()`和`equalTo()`时，如果子节点的公有属性key包含多种数据类型，将按照这些函数的参数的类型排序，即只能返回这个类型的有序数据。上面的数据如果使用 `orderByChild('score').startAt(60).limitToFirst(4)` 将得到下面的结果：
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

<p style='color:red'><em>注意：如果path与value的总长度超过1000字节时，使用`orderByChild(key)`将搜索不到该数据。</em></p>

**orderByKey**
当使用`orderByKey()`对数据进行排序时，数据将会按照字典顺序排列。注意，节点名只能是字符串类型。

**orderByValue**

<!--
当使用`orderByValue()`时，子节点将会按照它们的值进行排序。排序的规则与`orderByChild()`相同，唯一的区别是使用的是本节点的值，而不是节点下指定字段的值。
-->

当使用`orderByValue()`时，按照直接子节点的 value 进行排序。仅当 value 为单一的数据类型时，排序有意义。如果子节点包含多种数据类型时，排序不固定，此时不建议使用`orderByValue()`获取全量数据，例如，
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
霸王龙的分数是 `string`类型，雷龙的分数是 `boolean` 类型，而其他恐龙的分数是 `numberic` 类型，此时使用 `orderByValue()` 获得全量数据时，是一个看似固定的排序结果；但是配合使用`limitToFirst()`时，将获得不确定的结果。`Object`类型数据的value值为null，不会出现在结果中。
当配合使用`startAt()`、`endAt()`和`equalTo()`时，如果子节点的value包含多种数据类型，将按照这些函数的参数的类型排序，即只能返回这个类型的有序数据。上面的数据如果使用```orderByValue().startAt(60).limitToFirst(4)```将得到下面的结果：
```json
{
    "linhenykus" : 80,
    "pterodactyl" : 93
}
```
<p style='color:red'><em>注意：如果path与value的总长度超过1000字节时，使用`orderByValue()`将搜索不到该数据。</em></p>

**orderByPriority**
当使用`orderByPriority()`对数据进行排序时，子节点数据将按照优先级和字段名进行排序。注意，优先级的值只能是数值型或字符串。

- １. 没有优先级的数据（默认）优先。

- ２. 接下来是优先级为数值型的子节点。它们按照优先级数值排序，由小到大。

- ３. 接下来是优先级为字符串的子节点。它们按照优先级的字典序排列。

- ４. 当多个子节点拥有相同的优先级时（包括没有优先级的情况），它们按照节点名排序。节点名可以转换为数值类型的子节点优先（数值排序），接下来是剩余的子节点（字典序排列）。

----