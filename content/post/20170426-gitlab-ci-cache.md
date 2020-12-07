---
title: gitlab-ci中pip缓存的配置
description: 最近给实验室的Gitlab服务器开启了CI功能，采用的是docker模式，每次都启动一个全新的镜像进行构建。
date: 2017-04-06
categories:
    - 开发
---


最近给实验室的Gitlab服务器开启了CI功能，采用的是docker模式，每次都启动一个全新的镜像进行构建。

为了对Python代码进行风格检查，每次在构建的时候，都需要先在启动的容器中使用pip安装flake8。由于每次构建之间的环境相互隔离，所以pip的缓存也就完全没有作用，每次都需要直接联网下载相关的包来安装。

<!--more-->

国内的网络大家都懂，下载新包的速度时好时坏，严重影响了构建的速度。在配置CI的时候，正好看到了有cache的选项，遂决定启用cache，不必每次都从网络下载。

首先根据网上查阅的资料，在.gitlab-ci.yml中配置如下：

```
image: python:3.6
cache:
  paths:
    - pip-cache
  key: $CI_PROJECT_ID
# This is a basic example for a gem or script which doesn't use
# services such as redis or postgres
before_script:
  - python -V # Print out python version for debugging
  - export PIP_CACHE_DIR="pip-cache"
  - mkdir -p pip-cache
  - ls pip-cache
  - pip install flake8
lint:
  script:
  - flake8 .
```

其中重点是cache部分，该部分paths置顶了要缓存的目录，key指定了缓存的key（即只有key匹配时，才会启用缓存）。

在这里，我使用了当前目录下的pip-cache目录作为pip的缓存目录，项目id作为key，也就是本项目的所有构建都会共享这个目录。


另一个重点是`export PIP_CACHE_DIR="pip-cache"`这条命令。这条命令设置了一个PIP_CACHE_DIR的环境变量，pip会根据这个环境变量，将缓存放在我们指定的pip-cache目录下。


配置完成后，启动pipeline运行，发现还是每次都会从网络上下载Python的包。


经过多次搜索后发现，还需要对gitlab-ci-multi-runner进行配置。

我是根据系统帮助，直接在Ubuntu仓库里安装的，因此配置文件在`/etc/gitlab-runner/config.toml`当中。

打开该文件，可以看到[runners.docker]部分中的`volumes = ["/cache"]`这一条配置。根据我们对docker的了解，如果要进行持久化，需要将外部的一个目录挂载到容器内部，但是这里明显没有指定外部的目录。

修改这一行为：

```
volumes = ["/root/build_cache:/cache:rw"]
```
也就是把外部的/root/build_cache目录挂载到容器中的/cache目录中，并且给予读写权限。这样gitlab-runner每次创建缓存的时候，都会在/cache中(默认配置，可以按照官网文档修改)，也就是存到了宿主机的/root/build_cache目录中。当启动一个新的容器的时候，也会从宿主机的/root/build_cache中加载缓存文件。

最后，重新启动pipeline，发现pip已经可以成功使用缓存安装需要的包了。