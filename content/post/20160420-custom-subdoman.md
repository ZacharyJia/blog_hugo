---
title: 使用泛域名解析和Laravel路由实现用户自定义子域名
date: 2016-04-20
slug: custom-subdomain-using-laravel
categories:
    - 开发
tags:
    - Laravel
---

前段时间看到有人给简书提供的建议里有一条是希望简书能够提供用户自定义子域名功能。作为一个攻城狮，自然就开始想到自己能够怎么实现这个功能，于是马上联想到了“泛域名解析”功能。再加上之前录制《Laravel 入门之路由》这门课程的时候，提到过的**子域名路由**这个功能，马上就想到了针对用户自定义域名的解决方案。

<!--more-->

首先呢，稍微解释一下**泛域名解析**，泛域名解析就是在添加域名的解析记录的时候，添加一条带通配符`*`的记录，这样就能够匹配到其他的所有域名。
比如，下图是DNSPod的域名解析服务，可以看到，它提示我如果使用`*`的话，就可以匹配其他所有域名。

![DNSpod的域名记录](http://upload-images.jianshu.io/upload_images/66827-8d0b8c22a51d180e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

所以，我们就添加这样的一条记录。
![添加的记录](http://upload-images.jianshu.io/upload_images/66827-a52eb6ac727c6d48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里因为我在内网调试，所以就直接把记录的内容填成了我的内网地址。大家在使用的时候记得填写服务器地址就行。

等待一会儿之后，添加的记录就生效了。这时候，我只要随便输入一个之前不存在的子域名，都会指向192.168.1.101。比如：

![随意的子域名](http://upload-images.jianshu.io/upload_images/66827-fb134ad9bc04f0ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在，我们只需要进入到Laravel当中去，修改一下路由。

![修改路由](http://upload-images.jianshu.io/upload_images/66827-0a2331fb9594a8cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图里，我给这个组添加了一个domain的限制，并且在它对应的值里添加了一个user的参数，也就是说它会将子域名部分当做参数，传递给组内的所有请求处理函数。
然后，在里面的这个请求处理函数里，我只是简单的显示了一下这个子域名。对于大家来说，可以通过这个参数，找到对应的用户，显示用户的个人主页，这样就是可以实现通过子域名访问用户的主页啦！


![结果是这样滴](http://upload-images.jianshu.io/upload_images/66827-9debdf99e4f943ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当然，在这之前还要进行子域名和用户之间的绑定，不过这个就非常简单了，在数据库里添加一条记录就可以了。

这样，通过泛域名解析和Laravel提供的的路由功能，就能非常简单的实现用户的自定义域名啦。

PS：我这里只是进行了原理性的演示，实际使用过程中还需要对服务器软件（Apache、Nginx等）进行配置，让它们能够支持通过不同的域名来访问。

