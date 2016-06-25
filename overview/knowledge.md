/*
Title: 预备知识
Sort: 5
Tmpl : page
*/

## 术语

**appId** 作为野狗应用的唯一标识。创建野狗应用的时候必须设置一个appId。  

**URL** 统一资源定位， 参考文档 [RFC 1738](http://tools.ietf.org/pdf/rfc1738.pdf)。通过`appId`组成的 URL - `<appId>.wilddogio.com`，可以访问你的应用数据。

**path** 每个数据都有对应的path作为它的唯一标识。

**控制面板** 野狗官方提供的应用管理后台。

**JSON** 参考[网站 json.org](http://www.json.org)

**长连接** 连接建立以后不会断开，后续将复用这个连接发送数据，减少重复建立连接的开销。同时实现了全双工通讯，服务端可以做到实时向客户端推送数据。

**websocket** 通过 HTTP 协议与云端建立长连接，野狗的长连接就是基于 websocket 的。只有在握手过程时有 HTTP header，后续的数据的传输只有websocket的header，没有 HTTP header。参考 [RFC 6455](https://tools.ietf.org/pdf/rfc6455.pdf)。

**WSS** websocket + HTTPS，保障数据传输过程的安全性，以及起到云端的身份认证。

**HTTPS** HTTP + TLS，参考 [RFC 2818](http://tools.ietf.org/pdf/rfc2818.pdf)。

**TLS** 用于实现的 TCP 通道加密。参考 [RFC 5246](http://tools.ietf.org/pdf/rfc5246.pdf)。

**long-polling** 

**payload** 数据在网络传输过程中，除了协议头以外的 application 数据。

**session resume** TLS 握手优化，在连接重新建立后，减少 TLS 握手过程，通过 TLS sessionId 恢复 TLS 会话。参考 [RFC 5077](http://tools.ietf.org/pdf/rfc5077.pdf)。

**AES-NI** AES-NI 指令集，可以提升 AES 加密8倍的效率。 

**CoAP** 用于物联网的数据通讯协议，参考 [RFC 7252](http://tools.ietf.org/pdf/rfc7252.pdf)。

**规则表达式** 野狗提供的一套声明式的app数据读写权限控制方式。它的语法类似JSON，简单明了。

**超级密钥** 野狗的每个app都有自己的超级密钥，用于在集成用户已有的终端用户系统，为生成的token作签名校验，以保障安全。也可用于用户的服务端到野狗云的超级权限认证（使用超级密钥直接认证的终端为超级权限，不受规则表达式的限制）。

<hr>



