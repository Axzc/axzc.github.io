---
title: nginx的反向代理和负载均衡
date: 2019-09-21 15:34:07
categories: 笔记
tags: nginx
---

### 什么是正向代理和反向代理
之前在知乎上看已过一个类似的问题[反向代理为何叫反向代理？](https://www.zhihu.com/question/24723688),其中有两个例子.感觉特别通俗易懂.我就把他引用过来了.
#### 正向代理
> A同学在大众创业、万众创新的大时代背景下开启他的创业之路，目前他遇到的最大的一个问题就是启动资金，于是他决定去找马云爸爸借钱，可想而知，最后碰一鼻子灰回来了，情急之下，他想到一个办法，找关系开后门，经过一番消息打探，原来A同学的大学老师王老师是马云的同学，于是A同学找到王老师，托王老师帮忙去马云那借500万过来，当然最后事成了。不过马云并不知道这钱是A同学借的，马云是借给王老师的，最后由王老师转交给A同学。这里的王老师在这个过程中扮演了一个非常关键的角色，就是代理，也可以说是正向代理，王老师代替A同学办这件事，这个过程中，真正借钱的人是谁，马云是不知道的，这点非常关键.

在这里例子中 A同学 可以替换成 你正在使用的电脑 而老师就是代理服务器, 马云是要访问的目标.
<!--more-->
实际上在我们使用科学上网的时候使用的就是正想代理,比如 我们要访问 https://www.google.com 因为GFW 直接访问肯定是不行的.这时就需要在国外搭建一台代理服务器.让代理服务器去访问谷歌,最后再把结果返回到我们使用的电脑上.
#### 反向代理
>大家都有过这样的经历，拨打10086客服电话，可能一个地区的10086客服有几个或者几十个，你永远都不需要关心在电话那头的是哪一个，叫什么，男的，还是女的，漂亮的还是帅气的，你都不关心，你关心的是你的问题能不能得到专业的解答，你只需要拨通了10086的总机号码，电话那头总会有人会回答你，只是有时慢有时快而已。那么这里的10086总机号码就是我们说的反向代理。客户不知道真正提供服务人的是谁。

这里就可以理解为用户访问了一个网站,这个网站可能有成千上万的服务器,最后究竟是哪一个服务器处理用户的请求并且将处理请的请求返回给用户用户不需要知道,服务器只要把请求结果返回就行了,nginx就是一个性能很强的反向代理服务器.
而我们用到的反向代理通常就是  request > nginx > uWSGI

#### 负载均衡
负载均衡的意思就是在多台服务器中 nginx 作为调度者负责接受客户端的所有请求,再根据每一台uWSGI服务器的使用情况,将对应的请求交给uWSGI去处理.
负载均衡的五种策略

- 轮询(默认）
  把每个请求按照顺序逐一分配到不同的服务器.
  ```
  http {
    # ...
    upstream uwsgis {
        server 192.168.0.100:8080;
        server 192.168.0.101:8080;
        server example.com:8080;
    }
    server {
        listen 80;
        location / {
            proxy_pass http://uwsgis;
        }
    }
    # ...
}
  ```
proxy_pass http://tomcats; 表示吧所有请求转发到 uwsgis服务器组中的某一台服务器上
upstream模块：配置反向代理服务器组，Nginx会根据配置，将请求分发给组里的某一台服务器。uwsgis是服务器组的名称。

- 权重(weight)
```
upstream uwsgis {
    server 192.168.0.14 weight=3;
    server 192.168.0.15 weight=7;
}
```
weight 默认为1 将请求按照weight 分给每台服务器,如果有10次请求那么 14会分到3次, 15分到7次

- ip_hash
```
upstream uwsgi {
    ip_hash;
    server 192.168.0.14:88;
    server 192.168.0.15:80;
}
```
轮询和权都存在一个问题,就是不能保证用户的请求总会被一台服务器处理,那么一个已经登录的用户再通过上面两种方式定位到其他服务器上面就会丢失之前的登录信息,ip_hash会通过 *哈希算法*自动定位到已登录的服务器.
*每个请求按访问ip的hash结果分配，这样每个用户固定访问一个后端服务器，可以解决session的问题。*

- fair
按照服务器的响应时间来分配请求, 响应时间端的会优先分配
```
upstream uwsgis {
    server server1;
    server server2;
    fair;
}
```
- url_hash
 在upstream中加入hash语句，hash_method是使用的也是算法。
```
upstream uwsgis{
server 192.168.0.14:88;
server 192.168.0.15:80;
hash $request_uri;
hash_method crc32;
}
```
