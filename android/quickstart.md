/*
Title: 快速入门
Sort: 0
Tmpl : page-quick
*/

## 引入 SDK

#### 使用Maven获得 Android SDK：
```XML
<dependency>
	<groupId>com.wilddog</groupId>
	<artifactId>wilddog-client-android</artifactId>
	<version>0.6.3</version>
</dependency>
```

#### 使用Gradle获得 Android SDK：

要使用在 Android application使用 Gradle 或 Maven 添加 Wilddog 的依赖。
在你的build.gradle添加：

```
dependencies {
    compile 'com.wilddog:wilddog-client-android:0.6.3+'
}
```

如果出现由于文件重复的导致的编译错误，可以选择在build.grade中添加packingOptions：
```
android {
    ...
    packagingOptions {
        exclude 'META-INF/LICENSE'
        exclude 'META-INF/NOTICE'
    }
}
```

#### Java SDK：
在 Android SDK 之外，我们还有一个 Java 版本的 SDK，它不需要 Android 相关的依赖，可以通过 Maven 获得：
```XML
<dependency>
	<groupId>com.wilddog</groupId>
	<artifactId>wilddog-client-jvm</artifactId>
	<version>0.6.3</version>
</dependency>
```

<hr>

## Android 权限配置

Wilddog 在 Android 上需要 android.permission.INTERNET 权限。 你需要在 AndroidMainfest.xml 文件添加：

```XML
<uses-permission android:name="android.permission.INTERNET" />
```
----

## 初始化AndroidContext
在创建 Wilddog 实例之前，必须先进行一次初始化，设置AndroidContext。你可以在 `android.app.Application` 或者 `Activity`的`onCreate` 方法中设置 AndroidContext:

```Java
@Override
public void onCreate() {
    super.onCreate();
    Wilddog.setAndroidContext(this);
}
```

<hr>

##创建引用

需要先创建对 Wilddog 数据库的引用。这里会用到应用数据的 URL `https://<appId>.wilddogio.com/`。
```java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com/");
```

URL 中也可以包含数据路径，例如：
```java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com/users/Jack");
```
<hr>

## 写入数据

创建引用后，可以使用`setValue()`方法写入数据，支持以下类型:
`String`， `Boolean`， `Number`， `Map<String, Object>`。

```java
ref.setValue("hello world!!!");
```

<hr>

##读取数据
在野狗中，读取数据的方式是通过`addValueEventListener()`方法绑定一个 eventListener，用于对数据事件的处理：

```Java
ref.addValueEventListener(new ValueEventListener(){
	 public void onDataChange(DataSnapshot snapshot){
		 System.out.println(snapshot.getValue()); //打印结果 "hello world!!!"
	 }

	 public void onCancelled(WilddogError error){
		 if(error != null){
			 System.out.println(error.getCode());
		 }
	 }

});

```

上面的例子中，回调方法 onDataChange() 会在返回查询到的数据时调用一次，接下来在每次数据发生改变时再次被调用。

<hr>

## 数据同步

已经掌握了基本的数据读写能力，依赖这些我们可以实现数据实时同步的功能。我们可以做个demo去理解数据同步。多个客户端都使用`addValueEventListener()`函数，注册同一个`path`的`value`事件，其中一个客户端修改了数据，其余的客户端将触发`value`事件。这样就实现了一个数据同步的功能，一个客户端修改的数据，其他客户端通过事件的方式获得最新的数据状态。

这里我们准备了一个演示[数据同步的demo页面](https://cdn.wilddog.com/docs/demo/sync.html)，打开这个页面可以模拟同步效果。此页面就是使用`JavaScript SDK`实现的，它使用了`addValueEventListener()`函数注册了`value`事件。如下图：

![](https://cdn.wilddog.com/docs/demo/sync.png)

输入你的`<appId>`，点击关注后，最新的数据状态将在页面中的 textarea 里打印出来。如果你的应用当前没有数据，那textarea将是空的。然后，编写一个客户端程序，调用`setValue()`函数，同时观察页面上的打印，代码如下：

```Java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com");
ref.child('test_sync').setValue("hello world");
```

再使用`setValue()`重新设置一个值，同时观察页面上的打印，代码如下：

```Java
Wilddog ref = new Wilddog("https://<appId>.wilddogio.com");
ref.child('test_sync').setValue("hello world");
```

你会看到数据发生了变化。