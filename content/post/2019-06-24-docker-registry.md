---
layout:     post
title:      docker私有镜像仓库
subtitle:   工具配置
date:       2019-06-24
author:     axiomaster
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - docker
---

上一篇[VS code remote](http://www.axiomaster.com/2019/06/23/vscode-remote/)提到国内访问docker官方镜像特别慢，可以使用加速器。但对于企业等，都会自建私有镜像地址，也可以解决这个问题。

## docker registry

> Registry是一个无状态的，高可扩展的服务端应用，主要用于存储和分发Docker镜像。
>
>[Docker Registry文档](https://docs.docker.com/registry/)

### 下载registry镜像

1 直接运行registry

这种方式需要机器能够访问docker.io

```bash
docker run -d -p 5000:5000 --name registry registry:2
```

2 保存registry.tar并运行

- 通过可以访问外网的机器先 ```docker pull registry```放入本地镜像仓库

- 使用```docker save```保存为tar包

- 将registry.tar拷贝到无法访问外网的机器上

- 使用```docker load -i```将tar包加载到本地镜像仓库

- 最后```docker run```运行registry服务

### 配置私有镜像地址

1 将镜像```docker push```至私有镜像仓库

和使用registry.tar运行registry服务类似，思路如下：

- 将镜像tar包拷贝到机器上

- 使用```docker load -i```加载到本地镜像

- 使用```docker tag```为镜像打上私有镜像地址的标签，比如```192.168.1.11:5000/ubuntu:least```

- 使用```docker push```将镜像推到镜像仓库

### 使用私有镜像地址

修改```daemon.json```，添加如下配置：

```json
{
  "insecure-registries": ["192.168.1.11:5000"]
}
```

配置之后重启docker服务，就可以在Dockerfile中使用私有镜像仓库中的镜像了。
