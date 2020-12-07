---
title: 解决MySQL新建用户后无法登录问题
date: 2016-05-17
categories:
    - 开发
tags:
    - 数据库
    - MySQL
---
今天在给一台电脑配置XAMPP的时候，在上面的PHPMyAdmin里创建了一个新的用户，并且创建了密码，但是却一直无法使用这个账户登录到MySQL里。

<!--more-->

使用PHPMyAdmin的话，会提示登录失败。而直接在命令行登录的话，会提示`ERROR 1045 (28000): Access denied for user 'laravel'@'localhost' (using password: YES)`。如下图：

![使用命令行登录错误](http://qn-cdn.zacharyjia.me/mysql01.png)

按照网上说的解决方案，我尝试了使用root账户这个laravel账户进行授权，直接把all privileges授权给它，但是还是登录不成功。

最后，经过多次尝试后，我终于发现了问题所在：MySQL中默认存在一个用户名为空的账户，只要在本地，可以不用输入账号密码即可登录到MySQL中。而因为这个账户的存在，导致了使用密码登录无法正确登录。

![不用任何账号密码即可在本机登录进MySQL](http://qn-cdn.zacharyjia.me/mysql02.png)

解决方案：
只要通过root账户登录，然后将该账户删除即可：
```
mysql -u root   # 以root账户登录MySQL

use mysql   #选择mysql库

delete from user where User='';  #删除账号为空的行
flush privileges;  #刷新权限
exit  #退出mysql
```

现在，就可以使用刚才创建的账户和密码登录了：

![登录成功](http://qn-cdn.zacharyjia.me/mysql03.png)