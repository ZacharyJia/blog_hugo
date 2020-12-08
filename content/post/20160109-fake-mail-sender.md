---
title: 你真的知道谁给你发的邮件吗？——邮件发件人伪造
description: 如果发件人信息是可以伪造的，你还能知道是谁给你发的邮件吗？
slug: fake-mail-sender
date: 2016-01-09
categories:
    - 开发
tags:
    - 安全
---


一般来说，我们收到一封邮件之后，都会首先看发件人。有些重要的邮件，比如Apple给你发的重置密码邮件，手机丢失邮件等等，我们都会反复查验发件人，确认是真的来自Apple的官方，而不是钓鱼邮件。然而，如果这个发件人信息是可以伪造的，你还能知道是谁给你发的邮件吗？
前一段时间，学习SMTP邮件协议的时候，写了个简单的发件程序，却发现了一个很多邮箱都存在的发件人伪造问题。
想看伪造结果的，请直接拉到最下面看截图。


<!--more-->


## 0x00 邮件发送流程
在谈论问题之前，我们首先要了解邮件到底是怎么发送给对方的。下面这幅图是一个简单的发件流程：
![发件流程](http://qn-cdn.zacharyjia.me/imgSMTP1.png)
其中的Local Mail Server就是发件人的邮箱服务器，Remote Mail Server就是收件人的邮箱服务器。
发件人从Sender Main Client（可以是本地客户端程序, 比如foxmail、outlook，也可以是邮箱网页版）发送邮件之后，会首先通过SMTP协议到达Local Mail Server，然后由Local Mail Server同样通过SMTP协议将邮件投递到Remote Mail Server，之后收件人就可以在任意时间从自己的邮件服务器上收到邮件了。

## 0x01 SMTP协议
SMTP协议也就是简单邮件传输协议，上面我们说到，有两个地方都用到了SMTP协议。
那么我们接着就看看SMTP协议是怎么工作的（下面是两个终端之间建立TCP连接后传输的数据，S为接收方，C为发送方）：
```
S：220 163.com Anti-spam GT for Coremail System (163com[071018])
C：HELO smtp.163.com
S：250 OK
/*****************************
C：auth login
S：334 dXNlcm5hbWU6
C：USERbase64加密后的用户名
S：334 UGFzc3dvcmQ6
C：PASSbase64加密后的密码
S：235 Authentication successful
******************************/
C：MAIL FROM: <xxx@163.com>
S：250 Mail OK
C：RCPT TO:<XXX@163 .COM>
S：250 Mail OK
C：DATA
S：354 End data with .
C：From: xxx@163.com
C：To: xx@163.com
C：Subject: This is a test mail!
C：.
S：250 Mail OK
S：queued as smtp5,D9GowLArizfIFTpIxFX8AA==.41385S2
```
以上就是一个发送邮件的简单的例子,其中`***`之间的部分是从client到Local Mail Server这一步需要的，从Local Mail Server到Remote Mail Server是不需要的。

经过了上面的步骤，就可以将一封邮件发送到对方的邮箱了。

## 0x02 发件人伪造
知道了原理，我们就可以从这中间找破绽了。从上面的这段数据中，我们可以发现，关于发件人的部分一共有两个部分，一部分是
```
C：MAIL FROM: <xxx@163.com>
```
还有一部分在：
```
C：From: xxx@163.com
C：To: xx@163.com
C：Subject: This is a test mail!
```
其中，第一部分是通知邮件服务器发件人信息的，第二部分是邮件的具体内容，主要是在邮件的显示过程中，显示给用户看的，这部分可以随意修改，不会有影响。
真正有影响的是第一部分。这一部分是真正指示邮件发件人姓名的。有时候，修改了第二部分，邮箱会提示你，`该邮件由XXX代发`，其中的XXX就是真正的发件人。这说明有的邮箱会验证这两部分是否相同，如果不相同，就会提示用户。
而如果修改了第一部分会怎么样呢？

## 0x03 邮箱服务商的漏洞
为了能够伪造第一部分的收件人信息，我们采用的方法是，编写程序，模拟Local Mail Server，直接向对方的邮件服务器发送邮件。
这时，你可能会问，我们伪造的话，不会被对方发现吗？这就是部分邮箱服务商的漏洞所在！他们对于邮件的发送服务器并没有进行验证，而是无条件地信任了！也就是说，我说我是苹果的服务器，他就认为我是苹果的服务器！
正常来说，我们认为Remote Mail Server 收到邮件时，会进行反向验证，也就是你提供的发件人的邮件域名（@后面的部分）对应的MX记录与发件服务器的IP地址是否匹配，如果匹配的话，那么就认为没有问题，接收邮件；如果不匹配的话，就会拒收邮件。
而如果不进行认证的话，就可以随意进行伪造了。
我使用的程序是在计算机网络原理课上编写的一个小程序（伪造成本如此之低！）。


当过给一个有验证的服务器发送邮件的时候，结果是这样的(我交的服务器还不错)：
![bjtu](http://qn-cdn.zacharyjia.me/imgbjtu.png)

而如果我给一个没有验证的服务器发送邮件的时候，结果是这样的（没错，黑的就是你，QQ邮箱！）：
![qq邮箱](http://qn-cdn.zacharyjia.me/imgQQ.png)

然后我就会收到这样的邮件：
![QQ](http://qn-cdn.zacharyjia.me/imgQQ2.png)

嗯，如果把内容好好改一改，一封钓鱼邮件是不是就成功了呢^_^

当然，我是一个公平的人，怎么能只黑QQ邮箱呢，于是我把国内外几个常用的邮箱都试了一下，然后：
![126](http://qn-cdn.zacharyjia.me/img126.png)
![163](http://qn-cdn.zacharyjia.me/img163.png)
![sina](http://qn-cdn.zacharyjia.me/imgsina.png)
![outlook](http://qn-cdn.zacharyjia.me/imgoutlook.png)
嗯，看来不光是国内的问题，outlook也沦陷了。虽然有的被放到了垃圾邮件里，但是全部都收到了，而且并没有提示发件人有什么问题哦。

当然，还有一个没沦陷的，Gmail
![Gmail](http://qn-cdn.zacharyjia.me/imggmail.png)
这里提示的错误好像跟IPV6有关，但是我关闭IPV6之后，好像也被anti-spam系统拦下了，不知道有没有别的办法能突破，但是起码不是一下子就被突破了啊喂！

在我的测试中，只有我交的邮件服务器和Gmail没有发送成功，其他几个邮箱都收到了，其他还有多少邮箱存在这个问题我也不知道。

## 0x04 小结
这只是我在完成一个小作业的时候偶然发现的漏洞，当然，有可能各大邮箱服务商有自己的考虑，（比如兼容别的邮件服务器什么的？我不了解），所以没有进行反向验证。
但是伪造发件人的这个漏洞是的的确确存在的，也不排除有人会利用这个漏洞干坏事，大家以后还是要认真识别啊（好像并没有什么有效的方法啊。。。）