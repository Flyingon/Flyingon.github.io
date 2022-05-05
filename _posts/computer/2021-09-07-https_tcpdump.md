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

## 关注点
### 为什么需要3个随机数：
```
对于客户端：
当其生成了Pre-master secret之后，会结合原来的A、B随机数，用DH算法计算出一个master secret，紧接着根据这个master secret推导出hash secret和session secret。

对于服务端：
当其解密获得了Pre-master secret之后，会结合原来的A、B随机数，用DH算法计算出一个master secret，紧接着根据这个master secret推导出hash secret和session secret。

在客户端和服务端的master secret是依据三个随机数推导出来的，它是不会在网络上传输的，只有双方知道，不会有第三者知道。同时，客户端推导出来的session secret和hash secret与服务端也是完全一样的。

那么现在双方如果开始使用对称算法加密来进行通讯，使用哪个作为共享的密钥呢？过程是这样子的：

双方使用对称加密算法进行加密，用hash secret对HTTP报文做一次运算生成一个MAC，附在HTTP报文的后面，然后用session-secret加密所有数据（HTTP+MAC），然后发送。

接收方则先用session-secret解密数据，然后得到HTTP+MAC，再用相同的算法计算出自己的MAC，如果两个MAC相等，证明数据没有被篡改。

// MAC(Message Authentication Code)称为报文摘要，能够查知报文是否遭到篡改，从而保护报文的完整性。
```
```
管是客户端还是服务器，都需要随机数，这样生成的密钥才不会每次都一样。由于SSL协议中证书是静态的，因此十分有必要引入一种随机因素来保证协商出来的密钥的随机性。对于RSA密钥交换算法来说，pre-master secret本身就是一个随机数，再加上hello消息中的随机，三个随机数通过一个密钥导出器最终导出一个对称密钥。pre-master secret的存在在于SSL协议不信任每个主机都能产生完全随机的随机数，如果随机数不随机，那么pre-mastersecret就有可能被猜出来，那么仅适用pre-master secret作为密钥就不合适了，因此必须引入新的随机因素，那么客户端和服务器加上pre-master secret三个随机数一同生成的密钥就不容易被猜出了，一个伪随机可能完全不随机，可是是三个伪随机就十分接近随机了，每增加一个自由度，随机性增加的可不是一。
```

## 问题现象
### go get失败: 
```
x509: certificate has expired or is not yet valid: current time 2021-09-06T17:25:16+08:00 is after 2021-09-06T05:19:55Z
```

## 参考：
[https://datatracker.ietf.org/doc/rfc6101/](https://datatracker.ietf.org/doc/rfc6101/) <br/>

[https://blog.csdn.net/liwenbo_csu/article/details/76691420](https://blog.csdn.net/liwenbo_csu/article/details/76691420) <br/>
[https://blog.csdn.net/liangyihuai/article/details/53098482](https://blog.csdn.net/liangyihuai/article/details/53098482) <br/>
[https://blog.csdn.net/qq_31442743/article/details/116199453](https://blog.csdn.net/qq_31442743/article/details/116199453)