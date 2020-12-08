---
title: laravel 路由缓存
description: 今天在调试Laravel时，发现Routes.php中定义的路由不起作用了！
date: 2016-07-05
slug: laravel-routing-cache
categories:
    - 开发
tags:
    - Laravel
---


今天在调试Laravel时，发现Routes.php中定义的路由不起作用了！不管怎么修改，甚至删除，新修改的路由都不生效。

<!--more-->

使用命令行运行
```
php artisan route:list
```
查看路由时，也发现不管怎么修改都没有变动。

在经过多次查找时候，突然想到了路由缓存。于是赶紧查了一下Laravel的路由缓存的用法及其设置。最后发现确实是由于Laravel中的路由缓存造成的问题。

Laravel中的路由缓存是为了提高系统的运行效率，防止在大项目中每次都查询Routes.php文件查找路由列表。而路由缓存的使用方法也非常简单，一共就涉及以下两个命令：

```
//创建及更新路由缓存
php artisan route:cache

//清除路由缓存
php artisan route:clear
```

当要创建路由缓存时，就使用第一个命令，然后不管你再修改routes.php文件，都不会起作用。如果更新了routes.php文件中的路由，想要让其生效，那么再次执行以下第一条命令就可以了。

如果想要不再使用路由缓存，则直接执行第二条命令即可。