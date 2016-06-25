/*
Title: 高级特性
Sort: 3
Tmpl : page
*/

## 节点操作
节点是 SDK 的一个重要组件，数据的解析和组合全部要通过节点操作来完成。下面我们了解一下节点的数据格式以及一些基本操作，更多细节请参阅 API 文档。
### 节点数据格式
C / RTOS / OpenWRT SDK 和 Arduino SDK 的节点数据格式略有不同，下面分别进行阐述。
#### C / RTOS / OpenWRT 节点数据格式

野狗 SDK 采用树形结构组织，C / RTOS / OpenWRT SDK 使用类 JSON 格式（树型结构），能够和云端数据树互相转化。例如，我们在云端建立一组数据：

![](https://cdn.wilddog.com/z/iot/images/guide_2_1.png)

在 SDK 中，节点的格式为`Wilddog_Node_T`，接收到云端数据后，我们建立的节点树如下，与云端是一一对应的。其中，上下两层次间是父子关系，父节点只指向第一个子节点；同一层次则是兄弟关系，是一个双向链表：

![](https://cdn.wilddog.com/z/iot/images/guide_2_2.png)

节点支持的数据类型包括：字符串、二进制数组、整数、浮点型、布尔型、Object 。

#### Arduino 节点数据格式
Arduino SDK 采用 JSON 字符串形式组织节点数据，上面的节点数据在 Arduino SDK 中的格式为：

	{\"mydevice\":{\"Temperature\":{\"east\":24,\"north\":24,\"south\":24,\"west\":22}}}

Arduino SDK 的节点操作是字符串操作，因此不做赘述，只介绍 C / RTOS / OpenWRT SDK 的节点操作。

### 创建节点

#### 创建叶节点

创建叶节点可以直接调用节点 create 函数族中对应类型的创建函数，例如，我们想要创建一个 key 为 "wilddog"，value 为 UTF-8 字符串 "hello word" 的节点，调用如下：
```c
Wilddog_Node_T* node = wilddog_node_createUString("wilddog", "hello world");
```
#### 创建节点树

树的创建较上面更复杂，现在我们想要创建下面节点树：

	{
		"data" :
		{
			"pm25" : 60,
			"temperature" : -1,
			"humidity" : 0.3,
			"time" : "2016-01-01"
	    }
	}

从上面可以得出，这棵树根节点下面有一个 "data" 子节点，"data" 节点下又有4个叶节点。代码及注释如下：
```c
//创建根节点，该节点为一个对象，key 为 NULL 
Wilddog_Node_T *head = wilddog_node_createObject(NULL);

//创建子节点 "data"，同样为对象
Wilddog_Node_T *data = wilddog_node_createObject("data");

//创建叶节点 "pm25"，value 为 60，是一个数字
Wilddog_Node_T *pm25 = wilddog_node_createNum("pm25", 60);

//创建叶节点 "temperature"，value 为 -1，是一个数字
Wilddog_Node_T *temperature = wilddog_node_createNum("temperature", -1);

//创建叶节点 "humidity"，value 为 0.3，是一个浮点数
Wilddog_Node_T *humidity = wilddog_node_createFloat("humidity", 0.3);

//创建叶节点 "time"，value 为 "2016-01-01"，是一个 UTF-8 字符串
Wilddog_Node_T *time = wilddog_node_createUString("time", "2016-01-01");

//将叶节点插入到 "data" 节点中
wilddog_node_addChild(data, pm25);
wilddog_node_addChild(data, temperature);
wilddog_node_addChild(data, humidity);
wilddog_node_addChild(data, time);

//将 "data" 节点插入根节点中
wilddog_node_addChild(head, data);
```

自此，一颗节点树创建完成。

### 修改节点

节点的 key 和类型创建后无法修改，只能对 value 进行修改，API 为`wilddog_node_setValue()`。

### 删除节点

节点删除 API 为`wilddog_node_delete()`，将该节点以及其下面所有子节点删除。

<hr>

## 离线功能

#### 离线行为

当暂时失去网络连接的时候，Wilddog 应用仍然能正常工作，但如果到达设定的超时时间（默认10秒）后仍无法连接网络，除了数据同步事件，其他操作会触发超时错误的回调。一旦网络连接恢复，数据将会和云端保持同步。

#### 离线事件

在实时应用中，检测客户端是否断线是非常有必要的。例如，当用户的网络连接中断时，我们希望标记这个用户为“离线”状态。

Wilddog 提供了离线事件功能，使得客户端连接断开时，指定的数据被写入云数据库中。不论是客户端主动断开，还是意外的网络中断，甚至是客户端应用崩溃，这些数据写入动作都将会被执行。因此我们可以依靠这个功能，在用户离线的时候，做一些数据清理工作。Wilddog 支持的所有数据写入动作，包括 set，push，remove，都可以设置在离线事件中执行。

下面是一个例子，使用`wilddog_onDisconnectSetValue()`方法，在离线的时候写入数据：
```c
Wilddog_T ref = wilddog_initWithUrl("https://<appId>.wilddogio.com/disconnectmessage");
//当客户端连接中断时，写入一个字符串，key 为 NULL 是由于该字符串指向本节点，无需再次赋值。
Wilddog_Node_T *value = wilddog_node_createUString(NULL, "offline");
wilddog_onDisconnectSetValue(ref, value, callback, NULL);
```

离线事件是如何工作的

当进行了一个离线事件调用之后，这个事件将会被记录在云端。云端会监控每一个客户端的连接。如果发生了超时，或者客户端主动断开连接，云端就触发记录的离线事件。客户端可以通过回调方法，确保离线事件被云端正确记录了。
<hr>
## Auth 认证

我们在云端提供了规则表达式功能，用户可以自定义规则控制权限。SDK 也提供了 Auth 接口配合完成用户规则，接口定义如下：
```c
Wilddog_Return_T wilddog_auth
    (
    Wilddog_Str_T * p_host, 
    u8 *p_auth, 
    int len, 
    onAuthFunc onAuth, 
    void* args
    )
```
每个 host，即`<appId>.wilddogio.com`共用一个 Auth Key，通过调用`wilddog_auth()`接口，能够实现权限认证。Auth Key 可以采用超级密钥，其他获取方式正在开发中，敬请期待。

<hr>

## 配置 SDK

SDK 包含条件编译选项和用户参数，可对 SDK 进行配置(Arduino SDK不支持配置)。

#### 配置条件编译选项

Linux 和 Espressif 平台的编译选项在 make 时指定， WICED 平台的编译选项在 project/wiced/wiced.mk 中，MICO 平台则在工程的配置中。

	APP_SEC_TYPE : 加密方式，目前支持轻量级加密库 tinydtls、ARM 官方加密库 mbedtls 和无加密 nosec；

	PORT_TYPE : 运行的平台，目前支持 Linux 和 Espressif；

Linux 和 Espressif 平台在 make 时指定选项，进行不同的编译，如：

	make APP_SEC_TYPE=nosec PORT_TYPE=linux

在其他平台中，上面的宏在 Makefile 中指定。如 WICED 平台中，Wilddog SDK被嵌入WICED编译框架。因此条件编译选项在 SDK 目录下的`project/wiced/wiced.mk`中，配置项和 Linux 平台中相似，`PORT_TYPE`设置为`wiced`。

#### 配置用户参数

用户参数在SDK 的 include 目录下 wilddog_config.h 中，包含如下参数：

`WILDDOG_LITTLE_ENDIAN` : 目标机字节序，如果为小端则该宏定义的值为1；

`WILDDOG_MACHINE_BITS` : 目标机位数，可为8/16/32/64；

`WILDDOG_PROTO_MAXSIZE` : 应用层协议数据包最大长度，其范围为0~1300；

`WILDDOG_REQ_QUEUE_NUM` : 请求队列的长度；

`WILDDOG_RETRANSMITE_TIME` : 单次请求超时时间，单位为ms，超过该值没有收到服务端回应则触发回调函数,并返回超时。返回码参见`Wilddog_Return_T`；

`WILDDOG_RECEIVE_TIMEOUT` : 接收数据最大等待时间，单位为ms。

<hr>

## 移植 SDK

C/嵌入式 SDK 可以很容易的被移植到各个平台上。本文档以 WICED 平台此为例，说明如何移植 SDK，其他已移植平台可以参考 SDK 的 docs 目录下对应的文档，未移植平台则可以参考下面步骤。

#### 将SDK拷贝到目标位置

首先，将SDK解压，并拷贝到`WICED-SDK-3.1.2\WICED-SDK\apps`中，即SDK位于`WICED-SDK-3.1.2\WICED-SDK\apps\wilddog-client-c\`，注意，由于 WICED 平台不支持带 `-` 符号的文件夹，因此需要将 SDK 目录名字修改，这里修改成`wilddog_client_coap`。

WICED 平台采用 WICED IDE，打开 WICED IDE，能够在工程下的`apps`目录下找到我们的  SDK。

![](https://cdn.wilddog.com/z/iot/images/wiced-wilddog.png)

#### 移植条件编译选项

WICED 平台需要用户为自己的 APP 编写 Makefile，且格式有严格要求，Makefile文件名称的前缀必须与目录名相同，以我们的例子为例，如下图：

![](https://cdn.wilddog.com/z/iot/images/wiced-make.png)

在`project/wiced/wiced.mk`中添加编译选项，并补完 Makefile，详见`wiced.mk`文件。

注意：如果你的平台不支持自定义 Makefile，那么请根据条件编译选项，将你所需的文件拷贝到项目中，避免出现重定义。需要选择拷贝的路径有：

`APP_PROTO_TYPE` : src/networking 目录下，根据编译选项拷贝文件夹以及其中的文件；

`APP_SEC_TYPE` ： src/secure 目录下，根据编译选项拷贝文件夹以及其中的文件，同时，增加全局宏定义 WILDDOG_PORT ，根据选用的加密方式不同值不相同，nosec 时 WILDDOG\_PORT值设为5683,否则设为 5684；

`PORT_TYPE` ： platform 目录下，根据编译选项拷贝文件夹以及其中的文件，如果你的平台不属于`linux`或`wiced`等已支持平台，那么你需要自己实现平台相关的函数接口。

#### 实现平台相关代码

需要实现的平台相关函数接口位于include/wilddog_port.h，如下：
```c
int wilddog_gethostbyname(Wilddog_Address_T* addr,char* host);

int wilddog_openSocket(int* socketId);

int wilddog_closeSocket(int socketId);

int wilddog_send(int socketId,Wilddog_Address_T*,void* tosend,s32 tosendLength);

int wilddog_receive(int socketId,Wilddog_Address_T*,void* toreceive,s32 toreceiveLength, s32 timeout);
```
注意：tinydtls 以及 mbedtls 因为涉及底层，如果需要移植，可能要针对这两个库进行一些调试工作。

