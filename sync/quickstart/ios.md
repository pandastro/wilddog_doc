title:  iOS 快速入门
---

## 第一步 SDK 导入

SDK 的导入方式有两种，您可以选择下面方式的其中一种：

###第一种：使用 CocoaPods 
要将 Wilddog SDK 导入到你的工程中，推荐使用 [CocoaPods](https://cocoapods.org/)，如果没用过 CocoaPods，请先访问 [CocoaPods getting started](https://guides.cocoapods.org/using/getting-started.html)。 


打开工程目录，新建一个 Podfile 文件

	$ cd your-project-directory
	$ pod init
	$ open -a Xcode Podfile # opens your Podfile in XCode

然后在 Podfile 文件中添加以下语句

	pod 'Wilddog'
	
最后安装 SDK

	$ pod install
	$ open your-project.xcworkspace
	
###第二种：手动集成 

1、下载 SDK。[下载地址](https://www.wilddog.com/download#ios)         
2、把 Wilddog.Framework 拖到工程目录中。  
3、选中 Copy items if needed 、Create Groups，点击 Finish。  
4、点击工程文件 -> TARGETS -> General，在 Linked Frameworks and Libraries 选项中点击 '+'，将 JavaScriptCore.framework、 libsqlite3 加入列表中。

## 第二步 开始开发应用

### 1、引入头文件

Objective-C 

	#import <Wilddog/Wilddog.h>


Swift

	import Wilddog

### 2、初始化

Objective-C 

```
// 创建一个引用到我们的数据库
Wilddog *myRootRef = [[Wilddog alloc] initWithUrl:@"https://<appId>.wilddogio.com"];
```

Swift

```
// 创建一个引用到我们的数据库
var myRootRef = Wilddog(url:"https://<appId>.wilddogio.com")
```

## 第三步 读写数据

### 1、写数据

数据写到 Wilddog 数据库是比较简单的。我们可以使用 `setValue` 写入任何合法的 JSON 数据。


Objective-C 

```
// 写数据
[myRootRef setValue:@"Do you have data? You'll love Wilddog."];

```

Swift

```
// 写数据
myRootRef.setValue("Do you have data? You'll love Wilddog.")

```


### 2、读数据

Wilddog 服务器把数据实时同步给每一个正在监听的客户端。我们用 observeEventType 方法监听数据变化，然后会有一个 block 回调一个 WDataSnapshot 对象，它里面包含我们所需数据。

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
  print("\(snapshot.key) -> \(snapshot.value)")
})

```

上述例子中，监听 value 这个事件，在初次获取数据的时候回调会触发一次，此后每当 myRootRef 节点下面的数据发生改变，回调都会被触发。了解关于更多的事件类型和如何处理事件数据，请参见 [查询数据](guide/4) 。

