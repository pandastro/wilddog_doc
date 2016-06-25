/*
Title : 用户认证
Sort : 2
Tmpl : page
*/

大多数应用程序需要知道用户的身份。开发者了解用户的身份就可以允许应用程序给用户提供个性化体验，并根据用户的身份授予不同的权限来访问他们的数据。证明用户身份的该过程被称为验证。Wilddog提供了一套完整的身份验证api。

当用户在Wilddog APP上认证，需要知道：

1、用户信息是通过服务器返回的。这样，您就可以通过用户信息自定义用户体验。

2、用户信息返回包含一个uid（唯一的ID）。uid 是一个包含你认证的提供者的名称，后跟一个冒号，和从提供者返回的一个唯一ID的字符串。

3、未认证用户 auth 信息是nil，而认证用户有一个唯一的auth.uid和其他用户相关信息。

<hr>

## 用户设置
1、打开控制面版  
2、选择你的APP  
3、选中“终端用户认证”栏  
4、选中“开启野狗默认用户数据库”  

#### 监测认证
要监听验证状态的变化，可以用observeAuthEventWithBlock方法。如果当前用户没有验证，authData是nil。

Objective-C

```
Wilddog *ref = [[Wilddog alloc] initWithUrl:@"https://<YOUR-WILDDOG-APP>.wilddogio.com"];

[ref observeAuthEventWithBlock:^(WAuthData *authData) {
    if (authData) {
        // 用户已认证
        NSLog(@"%@", authData);
    } else {
        // 用户未认证
    }
}];

```

Swift

```
let ref = Wilddog(url: "https://<YOUR-WILDDOG-APP>.wilddogio.com")

ref.observeAuthEventWithBlock({ authData in
    if authData != nil {
        // 用户已认证
        println(authData)
    } else {
        // 用户未认证
    }
})

```


你也可以调用 authData 方法去检测用户的认证状态。

Objective-C

```
Wilddog *ref = [[Wilddog alloc] initWithUrl:@"https://<YOUR-WILDDOG-APP>.wilddogio.com"];

if (ref.authData) {
    // 用户已认证
    NSLog(@"%@", ref.authData);
} else {
    // 用户未认证
}

```

Swift

```
let ref = Wilddog(url: "https://<YOUR-WILDDOG-APP>.wilddogio.com")

if ref.authData != nil {
    // 用户已认证
    println(ref.authData)
} else {
    // 用户未认证
}

```
当添加认证事件的观察者，Wilddog将返回WilddogHandle。您可以使用removeAuthEventObserverWithHandle 方法来停止监听。

Objective-C

```
Wilddog *ref = [[Wilddog alloc] initWithUrl:@"https://<YOUR-WILDDOG-APP>.wilddogio.com"];
WilddogHandle *handle = [ref observeAuthEventWithBlock:^(WAuthData *authData) { ... }];
[ref removeAuthEventObserverWithHandle:handle];

```

Swift

```
let ref = Wilddog(url: "https://<YOUR-WILDDOG-APP>.wilddogio.com")
let handle = ref.observeAuthEventWithBlock({ authData in ... })
ref.removeAuthEventObserverWithHandle(handle)

```

#### 用户登录
使用 authUser:password:withCompletionBlock: 可以创建新的用户，新的用户在Wilddog的独立的用户系统中保存，并且每个App的用户系统之间是相互隔离的。

Objective-C

```
[ref authUser:@"jenny@example.com" password:@"correcthorsebatterystaple"
    withCompletionBlock:^(NSError *error, WAuthData *authData) {
    
    if (error) {
        // 登录时发生错误
    } else {
        // 登录成功，返回authData数据
    }
}];

```

Swift

```
ref.authUser("jenny@example.com", password: "correcthorsebatterystaple") {
    error, authData in
    if error != nil {
        // 登录时发生错误
    } else {
        // 登录成功，返回authData数据
    }
}

```
token 的有效期默认为24小时。你可以在控制面板中的“终端用户认证”中改变它。

#### 退出登录
用 unauth 方法退出登录，退出登录后原来的token失效。

Objective-C

```
[ref unauth];

```

Swift

```
ref.unauth()

```

#### 存储用户数据

Objective-C

```
[ref authUser:@"jenny@example.com" password:@"correcthorsebatterystaple"
    withCompletionBlock:^(NSError *error, WAuthData *authData) {
    
    if (error) {
        // 错误 :(
    } else {
        // 验证成功完成 :)

        // 登录的用户的唯一标识符
        NSLog(@"%@", authData.uid);

        // 创建一个新的用户词典，其中包含用户信息
        // 通过authData提供的参数
        NSDictionary *newUser = @{
            @"provider": authData.provider
        };

        // 创建一个“user”节点
        // 类似于下面的URL路径
        //  - https://<YOUR-WILDDOG-APP>.wilddogio.com/users/<uid>
        [[[ref childByAppendingPath:@"users"]
               childByAppendingPath:authData.uid] setValue:newUser];
    }
}];

```

Swift

```
ref.authUser("jenny@example.com", password:"correcthorsebatterystaple") {
    error, authData in
    if error != nil {
        // 错误 :(
    } else {
        // 验证成功完成 :)

        // 登录的用户的唯一标识符
        println(authData.uid)

        // 创建一个新的用户词典，其中包含用户信息
        // 通过authData提供的参数
        let newUser = [
            "provider": authData.provider,
            "displayName": authData.providerData["displayName"] as? NSString as? String]

        // 创建一个“user”节点
        // 类似于下面的URL路径
        //  - https://<YOUR-WILDDOG-APP>.wilddogio.com/users/<uid>
        ref.childByAppendingPath("users")
           .childByAppendingPath(authData.uid).setValue(newUser)
    }
}
```

当用户使用上述代码保存在数据库中的用户节点，是这样的：

```
{
  "users": {
    "login:1": {
      "displayName": "alanisawesome",
      "provider": "password",
    },
    "login:2": {
      "displayName": "gracehop",
      "provider": "password",
    }
  }
}

```
<hr>

## 错误处理
当你的APP调用一些用户认证方法，结果会通过block传值过来。当用户在认证过程中出现错误，error同样被回调。

所有的NSError有code和description属性。

在某些情况下，你可能需要知道某种特定的错误来通知你的用户，以便提示他们重新登录。例如，如果你使用的电子邮件和密码验证，想检查一个无效的电子邮件或密码：

Objective-C

```
[ref authUser:@"jenny@example.com" password:@"correcthorsebatterystaple"
    withCompletionBlock:^(NSError* error, WAuthData* authData) {

    if (error != nil) {
        // 在登录时发生错误
        switch(error.code) {
            case WAuthenticationErrorUserDoesNotExist:
                // 无效的用户
                break;
            case WAuthenticationErrorInvalidEmail:
                // 无效的电子邮件
                break;
            case WAuthenticationErrorInvalidPassword:
                // 错误的密码
                break;
            default:
                break;
        }
    } else
        // 登录成功
    }
}];

```

Swift

```

ref.authUser("jenny@example.com", password: "correcthorsebatterystaple") {
    error, authData in
    if (error != nil) {
        // 在登录时发生错误
        if let errorCode = FAuthenticationError(rawValue: error.code) {
            switch (errorCode) {
            case .UserDoesNotExist:
                println("无效的用户")
            case .InvalidEmail:
                println("无效的电子邮件")
            case .InvalidPassword:
                println("错误的密码")
            default:
                println("其它错误")
            }
        }
    } else {
        // 登录成功
    }
}

```