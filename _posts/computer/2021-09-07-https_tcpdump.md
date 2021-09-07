---
layout: post
title: https协议流程分析
category: 计算机和OS
tags: [https, tcp, tls]
keywords: https, tcp, tls
---

## SSL握手流程

参考: [https://blog.csdn.net/liwenbo_csu/article/details/76691420](https://blog.csdn.net/liwenbo_csu/article/details/76691420)

![ssl_flow](/assets/img/ssl/ssl_flow.png)

流程：

- client hello
- server hello
- client key exchange
- change cipher spec


## 抓包分析： 

1. 三次握手，建立链接: 

![三次握手](/assets/img/tcp/connect.jpeg)

2. TLS成功: 

![ssl_suc](/assets/img/ssl/ssl_suc.jpeg)

3. TLS证书校验(客户端)失败：

go get现象: 
```
x509: certificate has expired or is not yet valid: current time 2021-09-06T17:25:16+08:00 is after 2021-09-06T05:19:55Z
```

![ssl_failed_cert](/assets/img/ssl/ssl_failed_cert.png)