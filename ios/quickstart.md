/*
Title: 快速入门
Sort: 0
Tmpl : page-quick
*/


## 引入 SDK

有两种方式可以引入 SDK：

#### 使用 CocoaPods 
野狗建议使用 [CocoaPods](https://cocoapods.org/) 管理工程的依赖。关于 CocoaPods 的入门，请参考： [CocoaPods getting started](https://guides.cocoapods.org/using/getting-started.html)。 


打开工程目录，新建一个 Podfile 文件

	$ cd your-project-directory
	$ pod init
	$ open -a Xcode Podfile # opens your Podfile in XCode

然后在 Podfile 文件中添加以下语句

	pod 'Wilddog'
	
最后安装 SDK

	$ pod install
	$ open your-project.xcworkspace
	
#### 手动集成 

1、下载 SDK。[下载地址](https://www.wilddog.com/download#ios)         
2、将 Wilddog.Framework 拖到工程目录中。  
3、选中 Copy items if needed 、Create Groups，点击 Finish。  
4、点击工程文件 -> TARGETS -> General，在 Linked Frameworks and Libraries 选项中点击 '+'，将 JavaScriptCore.framework、 libsqlite3 加入列表中。

<hr>

## 写入数据

在所需要的类中，引入头文件

Objective-C：

```
1、#import <Wilddog/Wilddog.h>

```

Swift：

```
1、import Wilddog

```

----


首先创建一个 Wilddog 的引用，需要传递一个数据路径对应的 URL 作为参数。接下来使用 setValue 写入数据：

Objective-C 

```
// 创建引用
Wilddog *myRootRef = [[Wilddog alloc] initWithUrl:@"https://<appId>.wilddogio.com"];
// 写数据
[myRootRef setValue:@"Do you have data? You'll love Wilddog."];

```

Swift
```
// 创建引用
var myRootRef = Wilddog(url:"https://<appId>.wilddogio.com")
// 写数据
myRootRef.setValue("Do you have data? You'll love Wilddog.")

```

<hr>

## 读取数据

Wilddog服务器把数据实时同步给每一个正在监听的客户端。我们用 observeEventType 方法监听数据变化，然后会有一个block回调一个WDataSnapshot对象，它里面包含我们所需数据。

Objective-C 

```
// 读数据并监听数据变化
[myRootRef observeEventType:WEventTypeValue withBlock:^(WDataSnapshot *snapshot) {
    NSLog(@"%@ -> %@", snapshot.key, snapshot.value);
}];

```

Swift
```
// 读数据并监听数据变化
myRootRef.observeEventType(.Value, withBlock: {
  snapshot in
  println("\(snapshot.key) -> \(snapshot.value)")
})

```

上述例子中，value 这个事件会在初次获取到数据的时候被触发一次，此后每当数据发生改变，都会被触发。


