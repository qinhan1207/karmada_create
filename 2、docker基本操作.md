## 	基本命令

```bash
# 查找镜像
$ docker search ngnix

# 下载镜像，如果未指定版本，默认下载最新版
$ docker pull nginx:1.26.0

# 删除镜像
$ docker rmi nginx:1.26.0

# 查看已下载镜像
$ docker image ls 

# 查看正在运行的容器
$ docker ps 
	-a # 查看所有容器

# 在后台启动一个镜像为 nginx:1.26.0 名字为 mynginx 的容器，且宿主机端口80映射容器内端口80
$ docker run -d --name mynginx -p 80:80 nginx:1.26.0
	-d # 后台运行
	--name # 镜像名称
	-p # 端口映射

# 启动容器
$ docker start nginx

# 停止
$ docker stop nginx

# 重启
$ docker restart ngnix

# 查看容器状态
$ docker stats nginx

	
# 查看日志
$ docker logs nginx

# 删除一个容器
$ docker rm -f mynginx

# 进入容器内部
$ docker exec -it mynginx /bin/bash

# 提交，将一个正在运行或已停止的容器的当前状态保存为一个新的镜像。它类似于对容器的文件系统进行快照，将容器内的更改保存下来，使你可以基于此创建新的容器实例。
$ docker commit -m "update index.html" mynginx mynginx:v1.0

# 保存，将镜像保存为一个tar文件
$ docker save -o mynginx.tar mynginx:v1.0

# 加载本地镜像 
$ docker load -i mynginx.tar 

# 目录挂载
$ docker run -d -p 80:80 -v /app/nghtml:/usr/share/nginx/html --name app01 ngnix

# 卷映射，卷默认存放在本机中的 /var/lib/docker/volumes 目录下
$ docker run -d -p 88:88 -v /app/nghtml:/usr/share/nginx/html -v ngconf:/etc/nginx --name app02 nginx

# 查看所有的卷
$ docker volume ls

# 查看容器的细节
$ docker container inspect app01

# 删除所有的container
$ docker rm -f $(docker ps -aq)


```

## 自己创建镜像并上传到社区

```bash
# 将容器mynginx此刻的状态保存为一个镜像，镜像名称为mynginx，版本号为v1.0
$ docker commit -m "update index.htlm" mynginx myngixn:v1.0

# 将刚才创建的镜像保存为一个tar文件
$ docker save -o mynginx.tar mynginx:v1.0

# 加载本地镜像
$ docker load -i mynginx.tar

# 登录到Dockerhub，密码为qinhan1104
$ docker login -u qinhan1207

# 将镜像推到社区之前需要先改标签
$ docker tag mynginx:v1.0 qinhan1207/mynginx:v1.0

# 推送
$ docker push qinhan1207/mynginx:v1.0

```

![image-20241111130214740](file://C:\Users\qinhan\AppData\Roaming\Typora\typora-user-images\image-20241111130214740.png?lastModify=1731302931)

## docker存储

```bash
# 目录挂载，以外边文件夹为准，将外部的目录挂载到容器内部的一个位置
# -v /app/nghtml:/usr/share/nginx/htlm
$ docker run -d -p 80:80 -v /app/nghtml:/usr/share/nginx/html --name app01 nginx:v1.26.0

# 卷映射	
# -v ngconf:/etc/nginx	docker会自动给这个卷创建一个存储位置去把容器内部的内容，即使在容器初始启动的时候，就跟内部的内容保持完全一致，卷默认存放在本机中的 /var/lib/docker/volumes 目录下
$ docker run -d -p 80:80 -v /app/nghtml:/usr/share/nginx/html -v ngconf:/etc/nginx --name app02 nginx
```

```bash
# 查看所有的卷
$ docker volume ls

# 手动创建卷
$ docker create haha

# 查看某个卷的详情
$ docker volume inspect ngconf
```

## docker网络

docker为每个容器分配唯一的ip，使用容器ip+容器端口可以相互访问

ip由于各种原因可能会发生变化

docker0默认不支持主机域名

mynet: 创建自定义网络，容器名就是稳定域名

```bash


# 创建一个自定义网络
$ docker network create mynet

# 查看
$ docker network ls  

# 运行容器时加入到自定义网络
$ docker run -d -p 88:80 --network mynet --name app01 nginx
 
```

## Docker Compose

![image-20241111154801031](C:\Users\qinhan\AppData\Roaming\Typora\typora-user-images\image-20241111154801031.png)

```bash
# 以后台方式上线
$ docker compose -f compose.yaml up -d
# docker compose 具有增量更新功能，当compose文件发生变化时，只需要重新使用该命令便可以自动进行判断，只更新修改的部分

# 下线，但是volumes不会消失，数据不会丢失
$ docker compose -f compose.yaml down
	- rmi all # 下线的同时移除所有镜像
	- v # 下线的同时移除卷


# 批量启动指定容器
$ docker compose start x1 x2 x3

# 停止
$ docker compose stop x1 x3

# 扩容
$ docker compose scale x2=3
```

上线指第一次创建应用并启动，start 是之前启动过，但已经停止

## Dockerfile（自己制作镜像）

![image-20241003202629919](C:\Users\qinhan\AppData\Roaming\Typora\typora-user-images\image-20241003202629919.png)

当构建了 Dockerfile 文件后，可以用以下命令来生成镜像

```bash
# 基于Dockerfile文件构建一个名字为 myjavaapp 的镜像，镜像版本为 v1.0，. 代表当前目录
$ docker build -f Dockerfile -t myjavaapp:v1.0 .
```

