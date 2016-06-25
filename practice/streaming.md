/*
Title : streaming
Sort : 3
Tmpl : page
*/


## Wilddog REST API Streaming

Wilddog REST API 也能支持 Streaming。如果你想为你的野狗应用添加实时通知功能，所在的平台还不支持，就可以考虑用[REST API](/rest/api#Streaming0)来实现。这里简要介绍下如何使用Streaming，然后会举一个java版本的实现示例。

### 三步实现 REST API Streaming

在 Wilddog 中,Streaming 对一个位置的变化等价于你添加监听的一个客户端相对于于云端的不同。Streaming 的实现用的是 [EventSource / Server-Sent Events ](http://www.w3.org/TR/eventsource/)，SSE是一个为从服务端推送过来的消息创建一个HTTP连接的API。

##### 第一步 设置请求头
设置Accept = "text/event-stream" 。
你还需要确保你的应用考虑到了HTTP重定向，特别是HTTP状态代码为307的情况。

##### 第二步 包含认证参数
如果你需要权限来读取数据，就必须包含认证参数。所有类型的请求和认证都支持auth参数，用于请求访问受安全和规则表达保护的数据。
该参数可以是你的超级密钥或认证token。如下：
```
curl \
https://samplechat.wilddogio.com/users/jack/name.json?auth=CREDENTIAL
```

##### 第三步 处理从服务器返回的数据
当监听到数据变化时，服务端将会以下面的协议来发送此事件
```
event: event name
data: JSON encoded data payload
```
这些消息的结构符合 EventSource 协议。服务端发送的时间类型分以下四种：put, patch, keep-alive, auth_revoked。
这里是一组从服务器返回的事件的一个例子：
```json
//  设置你整个数据
event: put
data: {"path": "/", "data": {"a": 1, "b": 2}}

//  推送key为c的新数据, 然后整个数据如下
// {"a": 1, "b": 2, "c": {"foo": true, "bar": false}}
event: put
data: {"path": "/c", "data": {"foo": true, "bar": false}}

// 在每个数据的key, 更新或添加数据，
// 然后整个数据如下
//  {"a": 1, "b": 2, "c": {"foo": 3, "bar": false, "baz": 4}}
event: patch
data: {"path": "/c", "data": {"foo": 3, "baz": 4}}
```
在此查看 [Streaming](/rest/api#Streaming0) 的完整文档。

### Streaming java版本示例
我们已经用 java 创建了一个简单的示例demo，供有在服务端使用 Wilddog REST 需求的用户参考。
demo源码在 coding 上，点击 [wilddog-sse-java](https://coding.net/u/wilddog/p/wilddog-sse-java/git) 获取。