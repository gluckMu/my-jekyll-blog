---
layout: post
title:  "iOS自签名证书单向认证"
date:   2017-04-06
desc: "iOS自签名证书单向认证"
keywords: "iOS开发"
categories: [iOS]
tags: [iOS]
---

# iOS自签名证书单向认证

## 服务器

### 证书生成

证书生成时候值得注意的是密钥最好在2048位以上，还有生成cer文件时，附上域名或者ip

* 生成key文件

`openssl genrsa -out server.key 2048`

* 生成cer文件

`openssl req -new -x509 -key server.key -out server.cer -days 365 -subj /CN=104.198.87.161`

CN后面替换成实际地址

* 生成csr文件

`openssl req -new -key server.key -out server.csr`

* 生成crt文件

`openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt`

## iOS客户端

### 关闭ATS

在plist文件中增加NSAppTransportSecurity配置，iOS 10以后多了些配置，可以自行查阅

### AFNetworking设置

>AFNetworking 3.5版本

* 将上一步服务器生成的cer文件放入工程中
* 设置AFNetworking中SessionManager的securityPolicy

```
AFHTTPSessionManager *manager = [AFHTTPSessionManager manager];
NSString *cerPath = [[NSBundle mainBundle] pathForResource:@"cert" ofType:@"cer"];
NSData *certData = [NSData dataWithContentsOfFile:cerPath];
NSSet *set = [[NSSet alloc] initWithObjects:certData, nil];
AFSecurityPolicy *securityPolicy = [AFSecurityPolicy policyWithPinningMode:AFSSLPinningModeCertificate withPinnedCertificates:set];
securityPolicy.allowInvalidCertificates = YES;
securityPolicy.validatesDomainName = YES;
manager.requestSerializer = [AFJSONRequestSerializer serializer];
manager.responseSerializer = [AFHTTPResponseSerializer serializer];
[manager setSecurityPolicy:securityPolicy];
```