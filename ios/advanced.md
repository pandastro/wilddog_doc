/*
Title : 高级特性
Sort : 3
Tmpl : page
*/


## 离线事件

#### 离线行为

在实时应用中，检测客户端是否断线是非常有必要的。例如，当一个用户的网络连接中断时，我们希望标记这个用户“离线”状态。

Wilddog 提供了离线事件功能，使得客户端连接断开时，指定的数据被写入云数据库中。不论是客户端主动断开，还是意外的网络中断，甚至是客户端应用崩溃，这些数据写入动作都将会被执行。因此我们可以依靠这个功能，在用户离线的时候，做一些数据清理工作。Wilddog 支持的所有数据写入动作，包括 setting,  updating, removing 都可以设置在离线事件中执行。

下面是一个例子，使用 onDisconnect 方法，在离线的时候写入数据：

Objective-C

```
Wilddog *presenceRef = [[Wilddog alloc] initWithUrl:@"https://samplechat.wilddogio.com/disconnectmessage"];
// 当客户端连接中断时，写入一个字符串
[presenceRef onDisconnectSetValue:@"I disconnected!"];

```

Swift

```
var presenceRef = Wilddog(url:"https://samplechat.wilddogio.com/disconnectmessage")
// 当客户端连接中断时，写入一个字符串
presenceRef.onDisconnectSetValue("I disconnected!")

```

**离线事件是如何工作的**

当进行了一个 onDisconnect 方法调用之后，这个事件将会被记录在云端。云端会监控每一个客户端的连接。如果发生了超时，或者客户端主动断开连接，云端就触发记录的离线事件。

客户端可以通过回调方法，确保离线事件被云端正确记录了：

Objective-C

```
[presenceRef onDisconnectRemoveValueWithCompletionBlock:^(NSError* error, Wilddog* ref) {
    if (error != nil) {
        NSLog(@"Could not establish onDisconnect event: %@", error);
    }
}];

```

Swift

```
ref.onDisconnectRemoveValueWithCompletionBlock({ error, ref in
    if error != nil {
        println("Could not establish onDisconnect event: \(error)")
    }
})

```

要取消一个离线事件，可以使用 cancel 方法：

Objective-C

```
[presenceRef onDisconnectSetValue:@"I disconnected"];
// 取消离线事件
[presenceRef cancelDisconnectOperations];

```

Swift

```
presenceRef.onDisconnectSetValue("I disconnected")
// 取消离线事件
presenceRef.cancelDisconnectOperations()

```

#### 检查连接状态

在许多应用场景下，客户端需要知道自己是否在线。Wilddog 客户端提供了一个特殊的数据地址：/.info/connected。每当客户端的连接状态发生改变时，这个地址的数据都会被更新。

Objective-C

```
Wilddog *connectedRef = [[Wilddog alloc] initWithUrl:@"https://samplechat.wilddogio.com/.info/connected"];
[connectedRef observeEventType:WEventTypeValue withBlock:^(WDataSnapshot *snapshot) {
  if([snapshot.value boolValue]) {
    NSLog(@"connected");
  } else {
    NSLog(@"not connected");
  }
}];

```

Swift

```
var connectedRef = Wilddog(url:"https://samplechat.wilddogio.com/.info/connected")
connectedRef.observeEventType(.Value, withBlock: { snapshot in
    let connected = snapshot.value as? Bool
    if connected != nil && connected! {
        println("connected")
    } else {
        println("not connected")
    }
})

```


/.info/connected 的值是 boolean 类型的，它只是一个本地连接状态的记录，并不是从云端读取的。

<hr>

## 云端时间戳

Wilddog 提供了一种将云端时间戳作为数据写入的机制。这个机制和 onDisconnect 方法组合起来，很容易实现记录客户端断线时间的功能：

Objective-C

```
Wilddog *userLastOnlineRef = [[Wilddog alloc] initWithUrl:@"https://samplechat.wilddogio.com/users/joe/lastOnline"];
[userLastOnlineRef onDisconnectSetValue:kWilddogServerValueTimestamp];

```

Swift

```
var userLastOnlineRef = Wilddog(url:"https://samplechat.wilddogio.com/users/joe/lastOnline")
userLastOnlineRef.onDisconnectSetValue([".sv": "timestamp"])

```



