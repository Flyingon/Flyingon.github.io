---
layout: post
title: websocket介绍
category: 基础知识
tags: [websocket]
keywords: websocket
---

### 链接

- 协议(RFC 6455): [https://tools.ietf.org/html/rfc6455#section-9.1](https://tools.ietf.org/html/rfc6455#section-9.1)

### websocket协议握手
#### 建立链接
判断浏览器是否支持原生WebSocket，只需要如下一行代码即可:
```
if(window.WebSocket){
   // 支持WebSocket
}
```
WebSocket的连接都基于HTTP请求，只需要在请求头当中包含一个Upgrade的请求头，这是向服务器指定某种传输协议。例如指定WebSocket协议就是:
```
// -client
// 浏览器发送一个请求到服务器，表示它想把HTTP协议转为WebSocket。客户端通过更新头字段（Upgrade header）实现了这个目的
GET /echo HTTP/1.1
Sec-WebSocket-Key: xx
Sec-WebSocket-Verson: xx
Connection: Upgrade
Upgrade: websocket

// -server
// 如果服务器识别WebSocket协议，它通过Upgrade header接受协议转换
Connection:Upgrade
Sec-WebSocket-Accept: xx
Upgrade: WebSocket
```
此时HTTP连接会被基于TCP/IP连接的WebSocket连接所取代。

WebSocket连接默认使用和HTTP(80)或者HTTPS(443)一样的端口，同样，你可以像部署Web服务一样使用其它端口

另外，WebSocket为了完成握手，服务器必须响应一个计算出来的键值。这个响应说明服务器理解WebSocket协议。就像暗号一样，只有对上暗号才是自己人。

如何计算响应键值的呢？很简单，响应函数从客户端的Sec-WebSocket-Key请求头中取得键值，并在Sec-WebSocket-Accept请求头中返回根据客户端通过SHA1计算出键值，通过BASE64返回字符串。
```
// -- node.js
// 计算响应键值函数
var KEY_SUFFIX = "";    //协议规范中包含的一个固定键值后缀，服务器必须得知道这个值
var hashWebSocketKey = function(key){
  var sha1_enc= crypto.createHash("sha1").update(key + KEY_SUFFIX, "utf8");
  return sha1_enc.digest("base64");
}
```
#### 关闭链接
WebSocket关闭时，不论客户端还是服务端都会发送一个终止的数字代码，以及一个表示关闭原因的字符串。
当然，这些就像上面关于帧( frame )内容提到的一样，关闭操作的数据边界依然以帧的形式传输，操作符(opcode)为8，其中包含终止代码和内容文本。
WebSocket数字代码的含义：

代码 | 描述 | 场景
---|---|---
0~999 | 禁止 | 1000以下代码皆无效，不用于任何目的
1000~2999 | 保留 | 这些代码都用于WebSocket的扩展和修订版本
3000~3999 | 需要注册 | 这些代码用于"应用程序、程序库、框架"，可在IANA( 互联网分配机构 )注册
4000~4999 | 私有 | 可以用作自定义

关闭代码正是在1000~2999之间，例如1000表示正常关闭，1001表示离开，1002表示协议错误，1007表示无效数据，1011表示意外情况等。

### 帧结构(frame)

![websocket_frame](/assets/img/basic/websocket_frame.png)

第二行: 十进制的数值依次排开，其实就是第几位(bit)。
1. FIN：WebSocket帧头第一个字节第一位( bit )表示FIN码，因为WebSocket可以多帧，当你需要多帧的时候，前面的所有帧FIN位设值为0，最后一帧设置为1，用来标识结束帧。
2. RSV1, RSV2, RSV3: 第一个字节第二位(bit)到第四位都是RSV码，分别占1bit，如果通信两端没有设置自定义协议，那么必须设置为0, 否则必须断掉WebSocket连接。
3. Opcode: 第一个字节的后四位是操作码(opcode)，这是一个十六位无符号整数。

操作码(16进制) | 操作码 | 消息类型
---|---|---
%x0 | 0 | 附加数据帧
%x1 | 1 | 文本
%x2 | 2 | 二进制数据
%x8 | 8 | 连接关闭
%x9 | 9 | ping
%xA | 10 | pong
--- | 其它 |一共16位，其余全部保留用作将来扩展
WebSocket以文本传输的时候，都为UTF-8编码，是WebSocket协议允许的唯一编码。

5. Mask: 第二个字节高1位(bit)为MASK掩码，俗称“屏蔽”，就是用来标识客户端到服务端的数据是否加密混淆内容(payload)。
6. Payload length: 第二个字节低7位( bit )用来标识消息内容的长度(payload len)。
7. Extended payload length: 上图的Extended payload length是用来标识扩展的长度。这样做的好处是使用可变位数来标示编码长度能使消息更加紧凑(7位、7+16位、或者7+64位)。
- 如果这个值以字节表示是0-125这个范围，则数据真实长度就是低7位的值。
- 如果这个值是126，则随后的两个字节表示的是一个16进制无符号数，用来表示传输数据的长度。
- 如果这个值是127,则随后的是8个字节表示的一个64位无符合数，这个数用来表示传输数据的长度。
8. Masking-key: 0或4个字节，客户端发送给服务端的数据，都是通过内嵌的一个32位值作为掩码的。掩码键只有在掩码位设置为1的时候存在，且在客户端到服务端发送消息时才会存在。那么当存在Masking-key时，服务器接收的每个数据包在处理之前都需要解除掩码。
9. Payload data: Payload数据。(x+y)位，负载数据为扩展数据及应用数据长度之和。
10. Extension data: x位，如果客户端与服务端之间没有特殊约定，那么扩展数据的长度始终为0，任何的扩展都必须指定扩展数据的长度，或者长度的计算方式，以及在握手时如何确定正确的握手方式。如果存在扩展数据，则扩展数据就会包括在负载数据的长度之内。
11. Application data: y位，任意的应用数据，放在扩展数据之后，应用数据的长度=负载数据的长度-扩展数据的长度。

由此也可看出，WebSocket最小的传输大小仅为2KB，譬如关闭( %x8 )帧。

![websocket_data](/assets/img/basic/websocket_data.png)
这个图表达更为形象，帧的所有内容几乎都被囊括其中。

### 会话层协议

关于WebSocket协议，与TCP一样可以异步发送消息，都是可以用作高级协议的传输层。

这么说不是把WebSocket协议等同于TCP，尽管把WebSocket当作传输层使用，它的层次仍然在TCP之上。

根据OSI的协议层次，IP在网络层，TCP/UDP在传输层，HTTP位于应用层，与WebSocket协议一样，同样有着帧实现的SPDY( 由Google提出，增量性的提高HTTP的性能 )，位于会话层。

下面是各个协议之间对比：

--- | TCP| HTTP| WebSocket
---|---|---|---
寻址 | IP地址+端口 | URL | URL
并发传输 | 全双工 | 半双工 | 全双工
内容 | 字节 | MIMI消息 | 文本或二进制
消息定界 | 否 | 是 | 是
消息定向 | 是 | 否 | 是
