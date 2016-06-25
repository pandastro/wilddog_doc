/*
Title: 读写数据
Sort: 2
Tmpl : page
*/

为了更好的理解数据读写，我们来实现一个简单的空气监测应用，一步步了解操作。这个应用的数据储存在这里：

	https://<appId>.wilddogio.com/c/saving-data/airmonitor

## 创建数据

首先我们创建一个 Wilddog 引用，Arduino SDK 是类 C++ 的实现，因此 API 略有不同：

- 1. C / RTOS / OpenWRT SDK
```c
Wilddog_T ref = wilddog_initWithUrl("https://<appId>.wilddogio.com/c/saving-data/airmonitor");
```
- 2. Arduino SDK
```c
Wilddog *ref = new Wilddog("https://<appId>.wilddogio.com/c/saving-data/airmonitor");
```

<hr>

## 写入数据

写入数据包含两个操作：set 和 push。它们的作用分别是：
* set：用于替换当前节点下的数据。
* push：在当前节点下新增数据，但数据的 key 随机生成 

set 是最基本的写数据操作，它设置当前指定数据节点的值，如果当前节点已经存在，旧值将被覆盖。我们已经创建了一个引用，假设从传感器中获取了温湿度和 PM2.5 的数据，接下来用 set 储存数据。

#### 创建节点树

将要建立的节点树如下，"pm25" 节点存储 PM 2.5的值，"temperature" 节点存储温度，"humidity"存储湿度（百分比），节点树的创建可参见高级特性中的“节点操作”文档。

	{
		"data" :
		{
			"pm25" : 60,
			"temperature" : -1,
			"humidity" : 0.3,
			"time" : "2016-01-01"
	    }
	}

#### 用 set 写入数据

假设函数 createTree 会返回我们需要的节点树。下面我们使用 set 将数据写入云端。

- 1. C / RTOS / OpenWRT SDK
```c
wilddog_setValue(ref, createTree(), callback, NULL);
```
- 2. Arduino SDK
```c
ref->setValue(createTree(), callback, NULL);
```

回调函数 callback 会在云端返回结果或者出错后被调用，在回调函数中，根据返回码能够知道写入是否成功。

#### 使用 push 保存一个列表
当多个用户同时试图在一个节点下新增子节点的时候，这时，数据就会被重写，覆盖。
为了解决这个问题，push 采用了生成唯一 ID 作为 key 的方式。通过这种方式，多个用户同时在一个节点下，或者同一用户在不同时刻 push 数据，key 一定是不同的。这个 key 是通过基于时间戳和随机的算法生成的。Wilddog 采用了足够多的位数保证唯一性。

你可能会需要定时向云端写入数据，同时将所有数据都保留起来而不是覆盖，那么我们可以采用 push 的方式，最终生成类似下面的节点树：

	{
		"random key 1" :
		{
			"pm25" : 170,
			"temperature" : -2,
			"humidity" : 0.5,
			"time" : "2016-01-02"
	    },
		"random key 2" :
		{
			"pm25" : 60,
			"temperature" : -1,
			"humidity" : 0.1,
			"time" : "2016-01-01"
	    },
		...
	}

调用 push 写入数据的代码如下：
- 1. C / RTOS / OpenWRT SDK
```c
wilddog_push(ref, createTree(), callback, NULL);
```
- 2. Arduino SDK
```c
ref->push(createTree(), callback, NULL);
```
回调函数 callback 会在云端返回结果或者出错后被调用，在回调函数中，根据返回码能够知道写入是否成功，同时，会将新生成的路径返回给你。

<hr>

## 读取数据

有时候我们可能需要读取用户的命令信息，如开关机。调用 get 操作，可以立即从云端读取数据。

调用 get 读取数据的代码如下：
- 1. C / RTOS / OpenWRT SDK
```c
wilddog_getValue(ref, callback, NULL);
```
- 2. Arduino SDK
```c
ref->getValue(callback, NULL);
```

回调函数 callback 会在云端返回结果或者出错后被调用，在回调函数中，根据返回码能够知道读取是否成功，同时，会将读取的数据镜像作为参数传递到回调函数中。

## 同步数据

大部分情况下，我们需要对用户的操作进行及时的反馈，循环读取数据显然不能满足我们的要求，我们需要实时同步数据。Wilddog 采用事件机制来同步数据，C/嵌入式 SDK 目前只提供一种数据事件：`value`，该事件用来读取当前节点的静态数据快照，在初次获取到数据时被触发一次，此后每当数据发生变化都会被触发。回调函数被执行时候，当前节点下所有数据的静态快照会被作为参数传入。调用 addObserver 操作，可以和云端同步数据。

调用 addObserver 同步数据的代码如下：
- 1. C / RTOS / OpenWRT SDK
```c
wilddog_addObserver(ref, WD_ET_VALUECHANGE, callback, NULL);
```
- 2. Arduino SDK
```c
ref->addObserver(WD_ET_VALUECHANGE, callback, NULL);
```

回调函数 callback 会在云端推送新数据或者出错后被调用，在回调函数中，根据返回码能够知道同步是否成功，同时，会将读取的数据镜像作为参数传递到回调函数中。

#### 取消同步

通过 removeObserver 方法可以取消一个事件回调函数的绑定：

- 1. C / RTOS / OpenWRT SDK
```c
wilddog_removeObserver(ref, WD_ET_VALUECHANGE);
```
- 2. Arduino SDK
```c
ref->removeObserver(WD_ET_VALUECHANGE);
```

