---
layout: post
title: https协议流程分析(ssl)
category: 计算机和通信
tags: [https, tcp, tls]
keywords: https, tcp, tls
---

## SSL握手流程

![ssl_flow](/assets/img/ssl/ssl_flow.png)

### 主要流程：

- client hello
- server hello
- certificate + server key exchange
- client key exchange
- new session + change cipher spec

## 流程(抓包分析)： 

### 1. 客户端发送ClientHello消息，内容包括：
```
 支持的协议版本，比如TLS1.0版，
 一个客户端生成的随机数（稍后用于生成“会话密钥”），
 支持的加密算法（如RSA公钥加密）
 支持的压缩算法
```

![client_hello](/assets/img/ssl/client_hello.png)

### 2. 服务端发送Server Hello消息：主要包含: 
```
服务器将其SSL版本号
加密设置参数
一个服务器生成的随机数（稍后用于生成“对话密钥”）
确认使用的加密方法（如RSA公钥加密）
```
![server_hello](/assets/img/ssl/server_hello.png)

### 3. 服务端发送证书(Certificate)，该证书用于向客户端确认服务器的身份。 同时包含:
Server Key Exchange
```
通过Diffie-Hellman算法来生成最终的密钥（也就是Sessionkey
pubkey: Diffie-Hellman算法中的一个参数，这个参数需要通过网络传给浏览器，即使它被截取也没有影响安全性 
```
Server Hello Done:
```
Server Hello 发送完毕
```

_如果配置服务器的SSL需要验证客户端的身份，会发送该消息（多数电子商务应用都需要服务器端身份验证。服务器如果需要验证客户端的身份，那么服务器会发一个“Certificate Request”给浏览器，而在很多实现中，服务器一般不需要验证客户端的身份）_

![certificate](/assets/img/ssl/certificate.png)

### 4. Client Key Exchange: 浏览器收到服务器发来的Certificate包来之后，运行Diffie-Hellman算法生成一个pubkey，然后发送给服务器。通过这一步和上面Certificate两个步骤，服务器和浏览器分别交换了pubkey，这样他们就可以分别生成了一个一样的sessionkey（具体的实现过程可以参考Diffie-Hellman算法的实现）
_如果你还不是很理解这个算法，那么这里你只需要知道服务器和浏览器向对方发送了pubkey之后，双方就可以计算出相同的密钥了，并且这个过程没有安全问题_

![client_key_exchange](/assets/img/ssl/client_key_exchange.png)

### 5. New Session: 完成上面的步骤，可以说TLS的握手阶段已经完成了。但是，服务器还会发送一个Session Ticket给浏览器。这个sessionticket包含
```
这个ticket的生命周期是两个小时（7200s）、它的id等等信息。
```
_这个sessionticket包有什么做用呢？握手阶段用来建立TLS连接。如果出于某种原因，对话中断，就需要重新握手。客户端只需发送一个服务器在上一次对话中发送过来的sessionticket。这个sessionticket是加密的，只有服务器才能解密，其中包括本次对话的主要信息，比如对话密钥和加密方法。当服务器收到sessionticket以后，解密后就不必重新生成对话密钥了。就可以继续使用上一次的连接了。_

![new_session](/assets/img/ssl/new_session.png)

### 6. 下面便开始传数据了。使用的是http协议，但数据被加密了。

![application_data.png](/assets/img/ssl/application_data.png)


## 常见问题
go get现象: 
```
x509: certificate has expired or is not yet valid: current time 2021-09-06T17:25:16+08:00 is after 2021-09-06T05:19:55Z
```

## 参考：

[https://blog.csdn.net/liwenbo_csu/article/details/76691420](https://blog.csdn.net/liwenbo_csu/article/details/76691420) <br/>
[https://blog.csdn.net/liangyihuai/article/details/53098482](https://blog.csdn.net/liangyihuai/article/details/53098482)