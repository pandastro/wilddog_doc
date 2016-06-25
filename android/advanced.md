/*
Title : 高级特性
Sort : 3
Tmpl : page
*/


## 离线事件

在实时应用中，检测客户端是否断线是非常有必要的。例如，当一个用户的网络连接中断时，我们希望标记这个用户“离线”状态。

Wilddog提供了离线事件功能，使得客户端连接断开时，指定的数据被写入云数据库中。不论是客户端主动断开，还是意外的网络中断，甚至是客户端应用崩溃，这些数据写入动作都将会被执行。因此我们可以依靠这个功能，在用户离线的时候，做一些数据清理工作。Wilddog支持的所有数据写入动作，包括set, update，remove，都可以设置在离线事件中执行。

下面是一个例子，使用`onDisconnect()`方法，在离线的时候写入数据：

```java
Wilddog presenceRef = new Wilddog("https://samplechat.wilddogio.com/disconnectmessage");
// 当客户端连接中断时，写入一个字符串
presenceRef.onDisconnect().setValue("I disconnected!");
```

**离线事件是如何工作的**
当进行了一个`onDisconnect()`调用之后，这个事件将会被记录在云端。云端会监控每一个客户端的连接。如果发生了超时，或者客户端主动断开连接，云端就触发记录的离线事件。

客户端可以通过回调方法，确保离线事件被云端正确记录了：

```java
presenceRef.onDisconnect().removeValue(new Wilddog.CompletionListener() {
    @Override
    public void onComplete(WilddogError error, Wilddog wilddog) {
        if (error != null) {
            System.out.println("could not establish onDisconnect event:" + error.getMessage());
        }
    }
});
```

要取消一个离线事件，可以使用`cancel()`方法：

```java
OnDisconnect onDisconnectRef = presenceRef.onDisconnect();
onDisconnectRef.setValue("I disconnected");
onDisconnectRef.cancel();
```

#### 检查连接状态
在许多应用场景下，客户端需要知道自己是否在线。Wilddog客户端提供了一个特殊的数据地址，即 `/.info/connected`。每当客户端的连接状态发生改变时，这个地址的数据都会被更新。

```java
Wilddog connectedRef = new Wilddog("https://samplechat.wilddogio.com/.info/connected");
connectedRef.addValueEventListener(new ValueEventListener() {
  @Override
  public void onDataChange(DataSnapshot snapshot) {
    boolean connected = snapshot.getValue(Boolean.class);
    if (connected) {
      System.out.println("connected");
    } else {
      System.out.println("not connected");
    }
  }
  @Override
  public void onCancelled(WilddogError error) {
    System.err.println("Listener was cancelled");
  }
});
```

`/.info/connected`的值是boolean类型的，它不会和云端进行同步。

<br>

## 云端时间戳

Wilddog提供了一种将云端时间戳作为数据写入的机制。这个机制和`onDisconnect()`方法组合起来，很容易实现记录客户端断线事件的功能：

```java
Wilddog userLastOnlineRef = new Wilddog("https://samplechat.wilddogio.com/users/joe/lastOnline");
userLastOnlineRef.onDisconnect().setValue(ServerValue.TIMESTAMP);
```

<br>

## Transaction

当处理可能被并发更新导致损坏的复杂数据时，比如增量计数器，我们提供了事务操作。事务操作需要提供两个参数：一个更新方法和一个可选的完成 callback 方法。更新方法提供当前数据，当前数据是云端读取的。举例说明，如果我们想在一个的博文上计算点赞的数量，我们可以这样写一个事务： 

```java
Wilddog upvotesRef = new Wilddog("https://docs-examples.wilddogio.com/android/saving-data/wildblog/posts/-JRHTHaIs-jNPLXOQivY/upvotes");

upvotesRef.runTransaction(new Transaction.Handler() {
    public Transaction.Result doTransaction(MutableData currentData) {
        if(currentData.getValue() == null) {
            currentData.setValue(1);
        } else {
            currentData.setValue((Long) currentData.getValue() + 1);
        }

        return Transaction.success(currentData); // 也可以中止事务 Transaction.abort()
    }

    public void onComplete(WilddogError wilddogError, boolean committed, DataSnapshot currentData) {
        // 事务完成后调用一次，获取事务完成的结果
    }
});
```

我们使用 currentData.getValue（）！= null  来判断计数器是否为空或者是自增加。
如果上面的代码没有使用事务, 当两个客户端在同时试图累加，那结果可能是为数字 1 且非数字 2。

注意：doTransaction() 可能被多次被调用，必须处理`currentData.getValue()`为 null 的情况。当执行事务时，云端有数据存在，但是本地可能没有缓存，此时`currentData.getValue()`为 null。

更多关于事务 [API 的文档](/android/api)。