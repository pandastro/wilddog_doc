/*
Title : 用户认证
Sort : 2
Tmpl : page
*/

绝大多数应用都需要一套终端用户账号体系。对终端用户进行唯一标识之后，才能为用户提供个性化的用户体验，控制用户对数据的访问权限。提供终端用户唯一标识的过程被称为终端用户认证。Wilddog为开发者提供了多种用户认证方式。

当一个终端用户进行认证之后，会有以下几件事情发生：

- １. 终端用户的信息通过回调方法返回给了客户端。这样应用就可以得到用户的信息。

- ２. 返回的用户信息之中包含一个唯一标识`uid`，它是按照一定的策略生成的，确保在多种终端用户登陆方式中可以保持唯一。并且对于特定的终端用户，这个id永远不会发生改变。`uid`字段是一个字符串，它包含终端认证的方式，和这种认证方式返回的id，中间用冒号分隔。

- ３. 在规则表达式中，内置对象auth被定义。对于未通过终端认证的用户，auth对象为null。对于通过终端认证的用户，auth对象包含终端用户的唯一标识(auth.uid)，可能还包含其他用户信息数据。

<br>

## 设置终端认证方式

下面是野狗支持的用户认证方式：

认证方式	| 描述
-------- | ---
自定义	| 你自行生成登录token，通过这种方式，可以集成你现有的用户账号体系。也可以用于服务器端和wilddog云进行交互。
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

- ３. 点击“点击设置白名单”链接，并将域名设置到“OAuth 跳转域名白名单”中。


#### 启动终端用户认证


- １. 进入`控制面板`。

- ２. 选择自己Wilddog App。

- ３. 点击`终端用户认证`。

- ４. 选中一个认证方式。

- ５. 选择以各种认证方式，并启用。

- ６. 如果你使用第三方社交平台帐号登录。需要将社交平台的 APP ID 和 APP Secret 添加到表单中。

#### 监控终端用户的认证状态
使用`AuthStateListener`，对终端用户的认证状态的改变进行监听。

``` java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com");
ref.addAuthStateListener(new Wilddog.AuthStateListener() {
    @Override
    public void onAuthStateChanged(AuthData authData) {
        if (authData != null) {
            // user is logged in
        } else {
            // user is not logged in
        }
    }
});
```

调用`getAuth()`方法，也可以判断用户的登陆状态。

```java
AuthData authData = ref.getAuth();
if (authData != null) {
  // user authenticated
} else {
  // no user authenticated
}
```

如果要停止对终端用户状态改变的监听，可以用`removeAuthEventListener()`方法：

```java
ref.removeAuthStateListener(authStateListener);
```


#### 终端用户登录

不同的终端认证方式，实现代码不同。但它们都有类似的签名，都接受一个`Wilddog.AuthResultHandler`的实现作为参数，用它的回调方法处理认证成功和错误结果。

``` java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com");
// 构造一个handler，用于处理终端用户认证的结果。
Wilddog.AuthResultHandler authResultHandler = new Wilddog.AuthResultHandler() {
    @Override
    public void onAuthenticated(AuthData authData) {
        // 认证成功
    }
    @Override
    public void onAuthenticationError(WilddogError error) {
        // 认证失败
    }
};

// 通过自定义token的方式进行终端用户认证
ref.authWithCustomToken("<token>", authResultHandler);

// 匿名认证
ref.authAnonymously(authResultHandler);

// Email/密码方式认证
ref.authWithPassword("jenny@example.com", "correcthorsebatterystaple", authResultHandler);

// 通过OAuth方式认证。 provider可以为weibo，weixin或qq等。
ref.authWithOAuthToken("<provider>", "<oauth-token>", authResultHandler);
```

终端用户认证生成的token有效期默认为24小时。


#### 终端用户登出

使用`unauth()`方法，可以使终端用户的token失效。用户将退出系统。

```js
ref.unauth();

```
<hr>

## 错误处理

调用认证方法时，传入一个`AuthResultHandler`的实现子类作为参数。当认证过程中发生任何错误时，它的`onAuthenticationError()`将会被调用。

有时候你需要根据特定的错误信息对用户进行提示。例如使用Email/Password认证方式时，你需要判断是否Email或密码错误：


``` java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com");
ref.authWithPassword("jenny@example.com", "correcthorsebatterystaple",
    new Wilddog.AuthResultHandler() {
    @Override
    public void onAuthenticated(AuthData authData) { /* ... */ }
    @Override
    public void onAuthenticationError(WilddogbaseError error) {
        switch (error.getCode()) {
            case WilddogError.INVALID_PASSWORD:
                break;
            default:
                break;
        }
    }
});

```


#### 错误列表


Error Code| Description
-------- | ---
AUTHENTICATION_DISABLED| 指定的认证方式在当前Wilddog应用下被禁用。
EMAIL_TAKEN | 无法注册用户，因为Email已经被使用。
INVALID_ARGUMENTS | 参数错误。
INVALID_CREDENTIALS | 提供用于OAuth认证的credential错误。可能是格式错误或已过期。
INVALID_EMAIL | Email地址或密码不合法。
PROVIDER_ERROR | 第三方平台错误。
UNKNOWN_ERROR | 未知错误。