---
title: Linux下安装Riverbed OPNET Modeler 18.6.1过程
date: 2018-01-16
slug: linux-opnet-installation-guide
categories:
    - 开发
tags:
    - OPNET
---

最近实验室要使用OPNET做仿真，但是无奈18.6在windows下的sitl接口一直有问题，没办法，只能尝试一下linux下的版本是否可以用。但是以前从来没在linux下装过OPNET，所以还是踩了一些坑，总结一下主要的安装步骤。

<!--more-->

## 0x00 安装环境
系统版本： Ubuntu 16.04 TLS Desktop （最新的版本可能要考虑一下gcc的版本问题，OPNET要求的gcc版本是4，但是好像16.04默认安装的版本也是可以的）
OPNET版本：18.6.1 build 20050

## 0x01 准备安装文件
linux下的OPNET安装文件是一个bin文件（modeler_1861_20050_linux86.bin），准备好放到系统里就行。

## 0x02 权限说明
因为我要使用sitl接口，所以整个的安装、运行全都是在root用户里进行的。如果不需要sitl接口，安装可以使用root用户，运行可以直接用普通用户。

## 0x03 安装步骤：
1. 启用root用户的图形界面权限

打开终端，在普通用户权限下运行
```
xhost +
```

2. 给安装程序赋予执行权限

进入到安装程序所在目录，执行
```
sudo chmod a+x modeler_1861_20050_linux86.bin
```

3. 使用root用户，启动安装程序

```
sudo su
./modeler_1861_20050_linux86.bin
```
然后应该可以看到图形界面，根据你的需要选择就可以，我这里全部下一步到安装完成。

4. 添加启动路径到PATH环境变量中

默认情况下，安装在`/usr/riverbed`文件夹中，可执行文件都在`/usr/riverbed/18.6/sys/unix/bin`目录下。
所以我们把这个路径添加到PATH变量中：
```
export PATH=$PATH:/usr/riverbed/18.6/sys/unix/bin
```
为了以后可以直接启动，也可以把这条命令添加到你的终端rc文件中，例如`~/.bashrc`或者`~/.zshrc`等

5. 安装csh

OPNET的启动依赖于csh，所以通过下面命令安装
```
sudo apt install csh
```

6. 启动测试

到这里安装就完成了，直接在终端执行
```
modeler
```
就可以启动OPNET了。

## 0x04 编译环境的配置
我这里默认已经装好了gcc和g++，也不需要配置其他环境变量，可以直接编译。如果系统里没有装的话，可以用下面命令安装
```
sudo apt install gcc g++
```

## 0x05 附赠添加启动快捷方式的步骤 (针对非root用户运行的情况)
```
sudo vim /usr/share/application/modeler.desktop
```
然后再打开的这个空文件里，按i进入insert模式，输入以下内容
```
[Desktop Entry]
Encoding=UTF-8
Name=RiverBed Modeler 18.6
Comment=riverbed modeler
Exec=/usr/riverbed/18.6/sys/unix/bin/modeler
Terminal=false
Type=Application
Categories=Application;Development;
```
按ESC退出insert模式，输入 `:wq` 保存退出即可。

这样在系统应用程序里，就可以看到这个程序了。


## 0x06 Trouble Shooting
我这里在编译链接的过程中，出现了下面的链接问题：
```
/usr/riverbed/18.6/sys/pc_intel_linux64/lib/libz.so.1: no version information available
```
初步判断是opnet自带的zlib的版本问题，所以我选择自己通过系统安装zlib1g，然后替换这个文件。步骤如下：
```
sudo apt install zlib1g
cd /usr/riverbed/18.6/sys/pc_intel_linux64/lib/
sudo mv libz.so.1 libz.so.1.backup
```
然后重新运行DES应该就可以了。


## 0x07 总结
以上就是在Ubuntu 16.04 上安装OPENT 18.6的过程，相对来说比windows上安装多需要一些经验，但是配置环境变量之类的情况要少很多。