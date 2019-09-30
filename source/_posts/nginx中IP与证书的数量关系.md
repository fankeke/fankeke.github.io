---
title: nginx中IP与证书的数量关系
date: 2019-09-30 20:50:04
tags:
---

## 前言

由于SSL协议在HTTP协议之上，当客户端和服务器建立ssl连接的时候，是在TCP之上建立的ssl连接，这时候由于服务器是不会收到host等http的字段，这时候如果是多vhost的服务器（如nginx），那么它就不知道传送哪个证书给客户端。所以有一种说法就是：一台单ip服务器只能支持配置一个证书[1][2]。不过HTTPS中的SNI扩展已经解决了这个问题，让单个IP的服务器能够支持任意多的域名证书。



## 一 SNI介绍

SNI是https的一个特征，它允许在建立ssl连接的时候，客户端发送请求的域名放在字段SNI中（server name identification.）这样服务器在建立SSL连接的时候，就能知道客户端请求的是哪个vhost，然后将相应的证书传递过去。


## 二 构建无SNI的环境

现在较高版本的nginx和openssl都默认支持了SNI扩展，想要复现无SNI的环境我可是花了一番功夫的。使用的nginx和openssl版本如下：
nginx-1.8.1
openssl-1.0.0d
(在高版本下的openssl下硬是没编译成功，老是报告编译错误。。）


 1 编译环境

![](/images/nginx中IP与证书的数量关系/1.png) 

其中最后一行是将SNI协议关闭，编译完成后查看是否已经关闭了tls：


![](/images/nginx中IP与证书的数量关系/2.png) 


ok,已经关闭了。


2 复现问题

在无SNI支持的环境下，我们来观察如果给多个vhost配置证书会发生怎样的情况。
配置如下：

![](/images/nginx中IP与证书的数量关系/3.png) 


生成nginx2和nginx4的证书私钥对（可以用自建CA后然后再签发，参考：openssl自建CA后颁发证书，这里就直接签发了)


![](/images/nginx中IP与证书的数量关系/4.png) 


![](/images/nginx中IP与证书的数量关系/5.png) 


3 验证证书

访问站点一：


![](/images/nginx中IP与证书的数量关系/6.png) 


访问站点二：



![](/images/nginx中IP与证书的数量关系/7.png) 


可以看到，两个站点都加载了同一个证书，即站点一的证书。


##  构建SNI环境

1 和构建无SNI环境类似，只需要将disable-tlsext改为enable-tlsext即可，观察：

![](/images/nginx中IP与证书的数量关系/8.png) 

ok,支持SNI了


2 配置站点后观察：
访问站点一：

![](/images/nginx中IP与证书的数量关系/9.png) 

访问站点二：

![](/images/nginx中IP与证书的数量关系/10.png) 

    
可以看到，站点二此时加载了自己的证书，即SNI生效了。


refer:
[1］http://www.111cn.net/sys/nginx/103081.htm
[2］http://www.ttlsa.com/web/multiple-https-host-nginx-with-a-ip-configuration/
