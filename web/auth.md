/*
Title : 用户认证
Sort : 2
Tmpl : page
*/

绝大多数应用都需要一套终端用户账号体系。对终端用户进行唯一标识之后，才能对用户进行个性化的用户体验，控制用户对数据的访问权限。提供终端用户唯一标识的过程被称为终端用户认证。Wilddog为开发者提供了多种用户认证方式。

当一个终端用户进行认证之后，会有以下几件事情发生：

- １. 终端用户的信息通过回调方法返回给了客户端。这样应用就可以得到用户的信息。
- ２. 返回的用户信息之中包含一个唯一标识`uid`，它是按照一定的策略生成的，确保在多种终端用户登陆方式中可以保持唯一。并且对于特定的终端用户，这个id永远不会发生改变。`uid`字段是一个字符串，它包含终端认证的方式，和这种认证方式返回的id，中间用冒号分隔。
- ３. 在规则表达式中，内置对象auth被定义。对于未进行终端认证的用户，auth对象为null。对于终端认证过的用户，auth对象包含终端用户的唯一标识(auth.uid)，可能还包含其他用户信息数据。

<hr>

## 设置终端认证方式

下面是wilddog支持的用户认证方式：

认证方式	| 描述
-------- | ---
自定义	| 你自行生成登陆token，通过这种方式，可以集成你现有的用户账号体系。也可以用于服务器端和wilddog云进行交互。
Email/Password| 由Wilddog对终端用户进行管理。终端用户通过Email和密码的方式注册和登录。
Anonymous	| 使用匿名认证后，系统会为每个匿名用户生成唯一的标识。在会话过程中保持唯一标识不变。
新浪微博	    | 使用新浪微博账户认证，只需要编写客户端代码即可。
微信        | 使用微信账户认证，只需要编写客户端代码即可。
QQ         | 使用QQ账户认证，只需要编写客户端代码即可。

----

#### 配置终端用户认证

出于安全的原因，如果你使用的是基于web的OAuth认证流程（新浪微博，微信，QQ），只有在你设置的白名单中的域名才可以进行终端用户认证。为了开发和测试的方便，所有Wilddog应用默认都将`localhost`和`127.0.0.1`加入了白名单。如果你将应用部署在其它的域名下，你需要将域名添加到白名单中。

- １. 进入应用的`控制面板`。
- ２. 打开`终端用户认证`。
- ３. 点击“OAuth 跳转域名白名单”进行相应设置。


#### 启动终端用户认证


- １. 进入`控制面板`。
- ２. 选择自己Wilddog App。
- ３. 点击`终端用户认证`。
- ４. 选中一个认证方式。
- ５. 选择以各种认证方式，并启用。
- ６. 如果你使用第三方社交平台帐号登录。需要将社交平台的 CLIENT ID 和 Secret 添加到表单中。
- ７. 如果使用微博帐号登录， 需要在微博开发平台配置回调地址。
在[微博开发平台](http://open.weibo.com/)，打开“我的应用”。点击“我的应用”，在左侧的菜单栏选中“接口管理” ->“授权机制"，修改“授权回调页”为https://auth.wilddog.com/v1/[WILDDOG-APPID]/auth/weibo/callback （注： 更换[WILDDOG-APPID]为你的WildDog App Id） 。
- ８. 如果使用QQ帐号登录， 需要在腾讯开发平台配置回调域名。
在[QQ互联](http://connect.qq.com/)，打开“[管理中心](http://connect.qq.com/manage/index)”菜单，在网站列表中选中应用。点击操作按钮，查看详情，在“我的应用”中点击“管理网页应用”。在“基本信息”的“回调地址”，填写 https://auth.wilddog.com/v1/[WILDDOG-APPID]/auth/qq/callback （注： 更换[WILDDOG-APPID]为你的Wilddog App Id）。
- ９. 如果使用微信帐号登录，需要在微信开放平台配置回调域名。
在[微信.开放平台](https://open.weixin.qq.com/)，打开“管理中心”， 选择“网站应用”。选中相关的应用，点击“查看”。在“网站信息”中配置回调域名 auth.wilddog.com。
- 10. 如果在微信App里使用微信帐号登录，需要在微信公众帐号配置回调域名。
在[微信.公众平台](https://mp.weixin.qq.com)，登陆公众平台。在左侧导航栏，选中“开发者中心”，在“接口权限表"中找到“网页服务” -> “网页账号” ->  “网页授权获取用户基本信息”后，点击“修改”，填写“授权回调页面域名” auth.wilddog.com。

#### 监控终端用户的认证状态

使用`onAuth()`方法，对终端用户的认证状态的改变进行监听。

``` js
// 编写一个回调方法
function authDataCallback(authData) {
  if (authData) {
    console.log("User " + authData.uid + " is logged in with " + authData.provider);
  } else {
    console.log("User is logged out");
  }
}
// 注册回调方法，在每次终端用户认证状态发生改变时，回调方法被执行。
var ref = new Wilddog("https://<appId>.wilddogio.com");
ref.onAuth(authDataCallback);
```

如果要停止对终端用户状态改变的监听，可以用`offAuth()`方法：

```js
ref.offAuth(authDataCallback);
```

可以使用`getAuth()`方法检查终端用户认证状态：

```js
var ref = new Wilddog("https://<appId>.wilddogio.com");
var authData = ref.getAuth();
if (authData) {
  console.log("User " + authData.uid + " is logged in with " + authData.provider);
} else {
  console.log("User is logged out");
}
```
<br>

#### 终端用户登录

根据终端认证方式的不同，这些接口接收的参数不同，但它们都有类似的签名，接受回调方法。

```js
// 创建一个回调来处理终端用户认证的结果
function authHandler(error, authData) {
  if (error) {
      console.log("Login Failed!", error);
  } else {
      console.log("Authenticated successfully with payload:", authData);
  }
}

//  通过一个自定义的Wilddog Token来认证用户
ref.authWithCustomToken("<token>", authHandler);

// 或者使用email/password认证方式。
ref.authWithPassword({
    email    : 'Loki@asgard.com',
    password : 'dwadwadc'
}, authHandler);

// 或者弹出OAuth认证，比如新浪微博
ref.authWithOAuthPopup("weibo", authHandler);

ref.authWithOAuthRedirect("weibo", authHandler);
```

终端用户认证生成的token有效期为24小时。


#### 终端用户登出

使用`unauth()`方法，可以使终端用户的token失效。用户将退出系统。

```js
ref.unauth();

```

#### 用弹出窗口和重定向的方式进行第三方用户认证

Wilddog的OAuth认证支持三种不同的方式：弹窗、浏览器重定向、OAuth token登录。

大多数浏览器阻止javascript弹出窗口，除非是用户行为触发的。因此，我们应该只在用户点击事件处理中调用authWithOAuthPopup()方法。

注意：浏览器的弹窗和重定向并不是在所有的浏览器环境下都可用。弹窗在iOS版的Chrome下，iOS的预览面板，和本地file://的url下不可用。因此，建议组合使用多种认证方式，确保任何环境下都没有问题。


```js
var ref = new Wilddog("https://<appId>.wilddogio.com");

// 优先选择弹窗方式，这样我们就不需要重定向网页了
ref.authWithOAuthPopup("weibo", function(error, authData) {

  if (error) {
    if (error.code === "TRANSPORT_UNAVAILABLE") {
      // 回退到浏览器重定向，当我们返回到原始页面的时候自动获取session。
      ref.authWithOAuthRedirect("weibo", function(error) { /* ... */ });
    }
  } else if (authData) {
    // 使用Wilddog的用户认证

  }

});
```
<br>

## 错误处理

调用的认证方法时，传入一个回调方法。这个方法被执行的时候，根据认证的结果，会传入一个`error`和`authData`对象作为参数。

所有`error`对象都至少包含一个`code`字段和一个`message`字段。有时还会有一些附加信息放在一个`details`字段中，例如：

```json
{
  code: "TRANSPORT_UNAVAILABLE",
  message: "There are no login transports available for the requested method.",
  details: "More details about the specific error here."
}

```

有时候你需要根据特定的错误信息对用户进行提示。例如使用Email/Password认证方式时，你需要判断是否Email或密码错误：


```js
var ref = new Wilddog("https://<appId>.wilddogio.com");
ref.authWithPassword({
  email    : 'Loki@asgard.com',
  password : 'dwadwa'
}, function(error, authData) {
  if (error) {
    switch (error.code) {
      case "INVALID_EMAIL":
        console.log("The specified user account email is invalid.");
        break;
      default:
        console.log("Error logging user in:", error);
    }
  } else {
    console.log("Authenticated successfully with payload:", authData);
  }
});
```
<br>

#### 错误列表

Error Code| Description
-------- | ---
AUTHENTICATION_DISABLED| 指定的认证方式在当前Wilddog应用下被禁用。
EMAIL_TAKEN | 无法注册用户，因为Email已经被使用。
INVALID_ARGUMENTS | 参数错误。
INVALID_CREDENTIALS | 提供用于OAuth认证的credential错误。可能是格式错误或已过期。
INVALID_EMAIL | Email地址或密码不合法。
INVALID_ORIGIN | 请求的来源域名不在白名单中。这是一个安全错误。
PROVIDER_ERROR | 第三方平台错误。
UNKNOWN_ERROR | 未知错误。

----