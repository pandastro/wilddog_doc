/*
Title: 快速入门
Sort: 0
Tmpl : page-quick
*/

## 引入 SDK

在HTML页面上使用 Wilddog JavaScript SDK，你只需要加入一个 script 标签：

```html
<script src = "https://cdn.wilddog.com/js/client/current/wilddog.js" ></script>
```

**Node.js 用户**

Wilddog Node.js 版 API 与 JavaScript 版完全一样。要在 Node.js 环境中使用 Wilddog SDK，可以通过 npm 安装 Wilddog 模块:

``` shell
 $npm install wilddog --save 
 
```

接下来`require` Wilddog 实例：

```
var Wilddog = require("wilddog");
```

**Typescript 用户**
用于 typescript 的 .d.ts 文件在这里：[https://github.com/WildDogTeam/wilddog-typescript](https://github.com/WildDogTeam/wilddog-typescript) 。

<br>

## 创建引用
要读写数据，必须先创建 Wilddog 引用：

```js
var ref = new Wilddog("https://<appId>.wilddogio.com/");

```
创建引用的时候，需要传入数据的 URL 做为参数。上面的代码定位到了数据库的根节点。URL 地址也可以包含数据路径，例如：

```js
var ref = new Wilddog("https://<appId>.wilddogio.com/users/Jack");

```

上面的代码定位到了数据库的`/users/Jack`节点上。

Wilddog 对象提供了许多用于读写数据的方法。例如通过`set()`、`update()`、`push()`、`remove()` 修改数据, 通过`on()`立即读取数据，并监听该节点数据的变化。

<hr>

## 写入数据

创建 Wilddog 引用之后，就可以通过`set()` 写入任何合法的JSON数据：

```js
ref.set({
    "name" : "Jack Bauer",
    "age" : 32,
    "location" : {
        "city" : "beijing",
        "zip" : 100000
    }
});
```
<hr>

## 读取数据

读取数据是通过绑定回调函数来实现的。假设我们按照上面的代码写入了数据，那么就可以使用`on()`函数来读取city字段的值。

```js
ref.child("location/city").on("value", function(datasnapshot,error) {
	if (error == null) {
		  console.log(datasnapshot.val());   // 结果会在 console 中打印出 "beijing"
	}
});
```

回调函数的参数是一个DataSnapshot对象类型，调用它的`val()`函数得到数据对象。上边这个例子中，`value`这个事件会在初次读取到数据的时候被触发一次，此后每当数据发生改变，都会被触发。

若要只读取一次，不在之后每次数据发生变化的时候触发回掉函数，可以使用`once()`函数替代`on()`函数。

<hr>

## 数据同步

已经掌握了基本的数据读写能力，依赖这些我们可以实现数据实时同步的功能。我们可以做个demo去理解数据同步。多个客户端都使用`on()`函数，注册同一个`path`的`value`事件，其中一个客户端修改了数据，其余的客户端将触发`value`事件。这样就实现了一个数据同步的功能，一个客户端修改的数据，其他客户端通过事件的方式获得最新的数据状态。

这里我们准备了一个演示[数据同步的demo页面](https://cdn.wilddog.com/docs/demo/sync.html)，打开这个页面可以模拟同步效果。此页面就是使用`JavaScript SDK`实现的，它使用了`on()`函数注册了`value`事件。如下图：

![](https://cdn.wilddog.com/docs/demo/sync.png)

输入你的`<appId>`，点击关注后，最新的数据状态将在页面中的 textarea 里打印出来。如果你的应用当前没有数据，那textarea将是空的。然后，编写一个客户端程序，调用`set()`函数，同时观察页面上的打印，代码如下：

```js
var ref = new Wilddog("https://<appId>.wilddogio.com");
ref.child('test_sync').set("hello world");
```

再使用`set()`重新设置一个值，同时观察页面上的打印，代码如下：

```js
var ref = new Wilddog("https://<appId>.wilddogio.com");
ref.child('test_sync').set("data sync, so easy");
```

你会看到数据发生了变化。