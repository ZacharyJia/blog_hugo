---
title: 修改Laravel自带的认证系统的User类的命名空间
date: 2016-09-19
slug: laravel-user-namespace
categories:
    - 开发
---



刚创建了一个新的Laravel 5.3项目，想要使用Laravel自带的认证功能。  

<!--more-->

但是我们都知道，Laravel默认情况下的Model都是放在app目录下的，也就是说其命名空间是`App`.但是有时候我们希望app目录能够更加整洁一点，所以想要把各个Model都统一放在Model目录下。
由于Laravel的app目录遵循了psr-4标准，也就是说会是用composer按照psr-4标准对各个类进行自动加载。如果我们直接修改目录，而不修改对应的命名空间的话，是无法正常加载这些Model类的。

所以，将User.php文件移动到了新的Model文件夹下的时候，需要同时将User类的namespace修改为`App\Model`。然后，需要执行
```
composer dumpautoload
```
命令，将修改后的类自动加载进来。


接着继续进行认证系统的创建。  
在执行了
```
php artisan make:auth
```
命令之后，在正常情况下，已经可以实现正常的注册、登录等功能了。

但是在修改完User的命名空间后，会发现出现了找不到User类的错误。我们刚才已经重新加载了User类，为什么还会出现找不到的问题？

仔细想想我们就会发现，由于登录、注册用到的代码都是Laravel框架自带的，默认情况下，它们会认为User类还在App命名空间下，所以登录的时候，会出现错误。

如何解决呢？  
在`config/auth.php`文件里，可以找到providers，在其中driver是eloquent的那一组中，可以看到model选项，默认为`App\User::class`，将其修改为`App\Model\User::class`即可。

这样应该就可以正常登录了。


所以总结一下，如果想要修改User的命名空间的话，需要以下几步：

1. 新建Model文件夹，移动User.php到该文件夹下
2. 修改User.php的namespace为`App\Model`
3. 执行`composer dumpautoload`，重新加载类
4. 将`config/auth.php`文件中的providers部分的model对应的类，修改为`App\Model\User::class`