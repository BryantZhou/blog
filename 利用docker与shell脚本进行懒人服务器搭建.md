# 利用docker与shell脚本进行懒人服务器搭建

---

## 前言

> 最近在学习后端的知识，自己搞了个自己玩的全栈项目，在需要发布上线的时候，一开始是在服务器用的是git拉代码->编译构造->重新启动的手动挡模式，发布次数频繁了或者前后端需要同时发布就觉得十分的繁琐，所以就想着怎么可以通过脚本就把所有过程自动化执行，让自己敲少几次命令

## 思路简介

> 阿里云有个CodePipeline服务，类似jenkins那样可以进行编译构建，不过到最后一步想部署容器的话，貌似需要开通集群之类的，但是钱要留着给女朋友花，所以就先放弃研究这一块的实现了...，然后只能想点比较简单暴力的方法去实现，结合自己的实际情况，一台笔记本走天下的，所以通过shell脚本来实现应该是比较合适的方案了

#### 原材料：

ssh shell docker node

#### 前端代码：

通过本地编译出dist文件用scp命令把文件放上去服务器对应的站点目录

#### 后端代码：

通过dockerFile编译出image上传到自己的仓库再触发服务器的对应脚本重新启动对应的container

#### 服务器: 

用的是Ubuntu 16.04，建立一个文件夹存着一系列对应项目的脚本，让本地通过ssh触发执行脚本

## 实现

> 使用docker是为了方便环境的构建，因为自己只有一个服务器，使用docker就可以很从容的部署以及管理多个项目，docker的规划如下图

<img src="https://course-wx-app.oss-cn-hangzhou.aliyuncs.com/e81d2b43c5114119baccb414cb4b60b9.jpeg" width="200" alt=""/>

### 服务器配置

##### 配置ssh免密登陆

1. 把本地的~/.ssh/id_rsa.pub复制到服务器~/.ssh/authorized_keys里面

2. 重新登陆一次即可

##### 安装docker以及存放脚本

1. [docker简介以及安装](https://docker_practice.gitee.io/) 如何使用下面会说
2. 在服务器喜欢的地方建一个存放执行脚本的文件夹，例如我就放在根目录下/server_sh，然后给个777权限

### 本机配置

1. 在本机创建一个文件夹server_sh，并给个777权限，同时创建一个存放阿里云服务器上面的脚本的文件夹aliyun，此文件夹对应服务器上面的server_sh
2. 在server_sh下面创建一个自动更新服务器文件update_aliyun.sh，通过scp命令把aliyun文件夹里面的shell脚本同步到服务器，只要修改了脚本，执行一次./update_aliyun.sh，即可完成同步

```
#!/bin/sh

#更新服务器脚本

scp -r ./aliyun/* root@(服务器ip):/server_sh/
```

3. 在aliyun文件夹建立首次运行docker的脚本、需要安装的数据库类型脚本，这里使用了nginx、mongodb、mysql

##### 注意1：db类型的镜像run的时候需要使用内部或者挂载主机目录[数据卷](https://docker_practice.gitee.io/data_management/bind-mounts.html)来存放db数据，不然容器被删除，数据就会丢失

##### 注意2：容器内容通讯需要建立一次内部网络，不然只能每个端口都暴露出去，我为了方便调试，所有的端口好都暴露了出来，从正式环境来讲，应该只暴露nginx的80端口即可，其他都使用容器[内部通讯](https://docker_practice.gitee.io/network/linking.html)

* docker_create.sh

```
#!/bin/sh

# 首次安装docker跑的脚本

# 创建docker内部网络
docker network create --subnet 192.168.0.0/16 local-network

#启动web服务
./webserver.sh

#启动mongodb数据库
./mongodb.sh

#启动mysql数据库
./mysql.sh

```

* webserver.sh(根据需要，由官方nginx为基础配置的镜像，后面会说)

```
#!/bin/sh

# 启动Nginx服务

#停止正在运行的镜像
docker container stop webserver

#删除旧镜像
docker container rm webserver

#删除image，确保启动的时候能拿到最新的镜像
docker image rm masonchow/webserver

#下载最新镜像并启动
docker run -d -p 80:80 --name webserver -v /dockerVolume/web:/web:ro masonchow/webserver

```

* mongodb.sh

```
#!/bin/sh

#停止正在运行的镜像
docker container stop mongo

#删除旧镜像
docker container rm mongo

#删除image，确保启动的时候能拿到最新的镜像，若不需要拿最新版本镜像则不用运行
docker image rm mongo

# 安装并启动mongodb
docker run -d -p 27017:27017 -v /dockerVolume/db/mongo/data:/data/db --name mongo --network local-network --ip 192.168.0.101 mongo
```
* mysql.sh

```
#!/bin/sh

#停止正在运行的镜像
docker container stop mysql

#删除旧镜像
docker container rm mysql

#删除image，确保启动的时候能拿到最新的镜像，若不需要拿最新版本镜像则不用运行
# docker image rm mysql

# 安装并启动mysql
docker run -d -p 3306:3306 -v /dockerVolume/db/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root --name mysql --network local-network --ip 192.168.0.100 mysql
```

4. 执行update_aliyun.sh同步脚本到服务器并且在服务器执行一次docker_create.sh之后，在服务器打`docker container ls`就可以看到正在运行的容器信息

### 项目配置

#### Nginx

> 因为只有一个服务器，如果多个前端项目的话，每次增加一个前端项目，就需要更改一次nginx的配置，所以就根据官方的nginx加以自己的配置，利用脚本把定制好的dockerfile构造一个image更新到自己的docker仓库上，给服务器下载使用即可

* DockerFile

```
FROM nginx:latest

#一些nginx配置
ADD conf/nginx.conf /etc/nginx/nginx.conf

#nginx站点配置
ADD conf/server_app.conf /etc/nginx/conf.d/default.conf

ADD conf/gzip.conf /etc/nginx/conf.d/gzip.conf

RUN cd / \ && mkdir /web 

WORKDIR /web

CMD ["nginx", "-g", "daemon off;"]
```

这里监听的web目录是docker监听的本地目录，是由于在上文启动nginx的时候跑了`-v /dockerVolume/web:/web:ro`

#### 前端项目

> 前端项目只需要在本地构建完之后把dist文件夹同步到服务器对应的文件夹即可，例如我在项目文件夹下会建立一个publsh.sh，同时给个777的权限即可

```
#!/bin/sh

#删除本地的依赖包
rm -rf ./node_modules

#重新安装
npm i

#开始构建
npm run build

#删除远端本身的文件
ssh root@120.78.190.53 "rm -rf /dockerVolume/web/项目文件夹"

#把本地编译的文件放上去远端
scp -r ./wallet root@服务器IP:/dockerVolume/web/
```

#### 后端项目

> 后端项目的思路是每次更新发布就跑一次项目的脚本，通过dockerflie把后端编译成一个新的image更新到自己的docker仓库让服务器的docker跑最新的镜像

* publish.sh

```
#开始构建
npm run build

#构建最新镜像
docker build -t 用户名/镜像名 .

#发布镜像
docker push 用户名/镜像名

# 删除构建之后的代码
rm -rf ./dist

# 服务器现实新镜像
ssh root@服务器IP "cd /server_sh && ./服务器上面对应的脚本.sh"
```


* DockerFile

```
FROM node:8.9.3

COPY /dist /server

WORKDIR /server

ENV NODE_ENV=production

COPY /package.json /server

RUN npm install

CMD ["node", "./app.js"]
```

* 服务器上面的脚本

```
#!/bin/sh

# 启动后端服务

#停止正在运行的镜像
docker container stop 容器名

#删除旧镜像
docker container rm 容器名

#删除image，确保下载的是最新镜像
docker image rm 用户名/镜像名

#下载最新镜像并启动
docker run -d -p 3005:3005 --name 容器名 --network local-network --ip 192.168.0.105 用户名/镜像名
```

---

### 最后

毕业一年多，第一次写文章，不足之出，请求指出，感谢
