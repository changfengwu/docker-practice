# Docker 活动总结

## 1. What's Docker
> Docker是一个开发，运输和运行应用程序的开放平台。Docker使您可以将应用程序与基础架构分离，以便快速交付软件。使用Docker，您可以像管理应用程序一样管理基础架构。通过利用Docker的方法快速发送，测试和部署代码，您可以显着减少编写代码和在生产中运行代码之间的延迟。

> Docker架构图:http://docker.paas.x/engine/docker-overview/#docker-architecture

Docker是用Go语言编写的,作为一种新的虚拟化方式,值得我们学习.


## 2. 活动体验

### 2.1 获取安装
其实Docker的安装非常方便,官方提供了多平台安装使用的解决方案.这次活动中可能比较花时间的就是关于Docker私有库等`daemon.json`配置上面.但其实官方也提供了详细的配置方式,所以,`just run it!`

### 2.2 初赛体验

初赛的题目其实很简单,目的其实是引导大家学习Docker的基本用法,为后面的理解Docker的设计基础和理解Docker是如何运行的打下基础.因为以前使用过一段时间Docker,所以初赛题其实就是最开始学习的时候的那些常用指令.

但是技术这个事情,光会使用命令是不够的,还得去理解,总结.一开始以为自己都会了,Docker其实也没什么其他的了,然而接下来就吃了亏.

### 2.3 决赛体验

决赛那天的题目下来的时候,有种似是非是的感觉.关于一些概念上的东西感觉有点不太清晰,时间也相当紧迫.当结束时才反应过来,原来自己对之前一直在用的Docker是模棱两可的.细心点可以发现,其实最后的题目完全是按照官方的教程一步一步引导走的.通过这次比赛,不经想到,学习一个新的东西,不应只是关注当时场景所需要的那几条命令,而是应该将眼光放得更远一些.

### 2.4 赛后反思

这次比赛后,关于registry私有仓库的搭建其实自己不是很清楚,借此机会学习总结了一波


### docker registry私有仓库搭建

> 获取registry镜像  
> 官网地址: https://hub.docker.com/_/registry/  
> 官网文档: https://docs.docker.com/registry/deploying/#start-the-registry-automatically

### 快速运行
```bash
# 启动注册表
$ docker run -d -p 5000:5000 --restart always --name registry registry:2

# 加入私有仓库
cat /etc/docker/daemon.json << EOF
{ "insecure-registries":["localhost:5000"] }
EOF

# 重启docker
systemctl daemon-reload
systemctl restart docker

# 使用
$ docker pull nginx
$ docker tag ubuntu localhost:5000/nginx:test
$ docker push localhost:5000/nginx:test

$ docker pull localhost:5000/nginx:test

# 停止并删除所有数据
$ docker container stop registry && docker container rm -v registry
```

### 启动配置
```bash
# 使用registry镜像生成密码
docker run --entrypoint htpasswd registry:2 -Bbn admin admin >>/home/xinchen/registry/auth/htpasswd


docker run -d \
  -e "REGISTRY_AUTH=htpasswd"  \        # 认证
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:5001 \ # 改变容器内部监听的端口
  -v /mnt/registry:/var/lib/registry \ # 自定义镜像存储位置
  -v /home/xinchen/registry/auth:/auth \ # 挂载用户名密码地址
  -p 5000:5001 \
  --name registry-test \
  registry:2


# 启动后需要登录
docker login localhost:5000
```

### API
> 官网: https://docs.docker.com/registry/spec/api/#detail

```bash
# 检测是否生效
curl -XGET 127.0.0.1:5000/v2/

# 获取当前仓库中有的镜像
curl -XGET 127.0.0.1:5000/v2/_catalog

# 获取仓库中指定镜像的tag list
curl -XGET 127.0.0.1:5000/v2/nginx/tags/list

# 检测镜像清单是否存在 
# HEAD /v2/<name>/manifests/<reference>
# reference可以是tag或者是digest
curl -I http://localhost:5000/v2/nginx/manifests/test

# 拉取镜像清单
# GET /v2/<name>/manifests/<reference>
curl -XGET http://localhost:5000/v2/nginx/manifests/test

# DELETE /v2/<name>/manifests/<reference> 删除镜像
```

### PULL/PUSH镜像过程
#### PULL镜像过程
> `image`是由`JSON manifest`(JSON清单) 和`individual layer files`(单个图层文件),`pull image`的过程围绕检索这两个组件

> `digest` 概要是镜像各个层的唯一标识。虽然算法允许使用任意算法，但是为了兼容性应该使用sha256。  
> 例：sha256:6c3c624b58dbbcd3c0dd82b4c53f04194d1247c6eebdaab7c610cf7d66709b3b

| field        | description   |  
| --------   | -----:  | 
| name     | The name of the image. |  
| tag        |   The tag for this version of the image.   |   
| fsLayers        |    A list of layer descriptors (including digest)    |  
|signature|A JWS used to verify the manifest content|

当获取清单之后，客户端需要验证签名`signature`，以确保名称和层是有效的。  
确认后，客户端可以使用`digest`去下载各个层，在`v2API`中，层存储在`blobs`中以`digest`作为键值。
```bash
# GET /v2/<name>/blobs/<digest>
curl -XGET http://localhost:5000/v2/nginx/blobs/sha256:f17d81b4b692f7e0d6c1176c86b81d9f2cb5ac5349703adca51c61debcfe413c
```

#### PUSH镜像过程
> 推镜像和拉取镜像过程相反，先推各个层到registry仓库，然后上传`manifest`(清单)。

```bash
# HEAD /v2/<name>/blobs/<digest>
# 返回200 ok则存在不用在推送
curl -I http://localhost:5000/v2/nginx/blobs/sha256:f17d81b4b692f7e0d6c1176c86b81d9f2cb5ac5349703adca51c61debcfe413c
```
