layout: next
title: docker
date: 2016-05-25 20:23:11
tags: docker
---

本文通过构建一个完整的基于nginx,nodejs,redis的web项目来介绍docker的开发和部署流程，希望对后来的人有帮助。<!--more-->
### 一：docker简介 
　　docker是一种能把应用和依赖打包到虚拟的容器中，并运行在任何Linux服务器上的工具。docker采用沙盒机制，保证所有的应用保持隔离。当应用被打包成docker image后，部署和运行变的极其的方便，这有助于应用的灵活性和便携性。docker主要可以分为container，image，docker hub等三部分。container是用户应用和服务真正运行的场所，可以简单的理解为一个个集装箱，用户的应用和服务就运行在container上，image是docker镜像，docker hub类似于github，github维护的是用户的代码，docker hub维护的是用户的镜像。用户可以把自己的image打包上传到docker hub,也可以在docker hub上下载其他用户或者官方的镜像。下面我们将会介绍基于docker搭建一个web服务。在本例子中我们将会实现一个统计页面访问次数并存入redis数据库的简单应用。该应用采用redis作为数据库，nodejs作为后台，nginx进行负载均衡。整个架构如下图所示：


### 二：docker安装

　　下面我们将会以64位的Ubuntu 14.04为例子介绍docker的安装过程。

* a:升级你的包管理器

 ``` bash
  $ sudo apt-get update
 ```

* b:安装所有必须和可选的包

 ``` bash
  $ sudo apt-get install linux-image-generic-lts-trusty
 ```

* c:重启电脑

``` bash
  $ sudo reboot
```

* d:获取最新的docker安装包

``` bash   
  $ curl -sSL http s: // get .docker. com / | sh
```

* e:运行docker守护进程

``` bash   
  $ sudo docker -d &
```

　　Note:发现运行过程中出现报错：“ubuntu 14.04 docker your kernel does not support cgroup memory limit: mountpoint for memory not found”。为了解决这个错误，首先打开grub文件，`sudo vi /etc/default/grub`，然后修改，`Modify: GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1”`，保存并推出，更新grub文件,`sudo update-grub`;最后重启电脑，`sudo reboot`。

* f:确认docker是否成功安装

 ``` bash    
  $ sudo docker run hello-world
 ```

如果正确安装，将会会提示“Hello from Docker.This message shows that your installation appears to be working correctly.”

* g：如果你不想每次运行docker使用root权限，可以进行一下操作

首先创建docker用户组并添加用户，`$ sudo usermod -aG docker ubuntu`；注销登录并重新登录；最后验证docker用户不再需要sudo命令执行docker：

```bash
 $ docker run hello-world
```

### 三：redis容器
  本小结将会介绍redis的安装和使用。

* a:下载最新的redis镜像

``` bash
 $ sudo docker pull redis:latest
```

我们直接从docker hub上下载最新的官方redis镜像。它预打包了redis的安装，默认端口是6379，你可以不用修改任何参数直接运行。

* b:启动redis容器，并将端口映射到默认的6379

``` bash
  $ sudo docker run -it —-name reids -p 6379:6379 redis
```

—-name参数可以指定容器的名字，-it表示容器以交互的形式运行，当然也可以使用-d让容器在后台运行，-p可以指定容器的映射端口，docker会将容器内部端口映射到本地host，如果采用-P，容器将会随机从49153 和65535之间查找一个未被占用的端口绑定到container。默认情况下，容器退出之后，容器将会保存下来，如果加上—rm参数，则docker会在container结束之后自动清理其产生的数据。

* c:数据的持久化
  
默认情况下容器运行期间产生的数据是不会写入镜像中，重新运行容器将会初始化数据。如果想要数据持久化可以使用数据卷挂载实现持久化。数据卷可以被不同的容器共享，容器可以直接对数据卷里面的内容进行修改，数据卷的变化不会影响容器的更新，即使容器被删除，数据卷也会一直存在。首先我们打开本地的redis.conf修改rdb文件的路径为：`dir ./`。然后将本地的rdb和config文件挂载到容器中的data目录下,本例子中rdb和config文件都在redis目录下：`-v /home/adventure/redis:/data`，最后启动:

``` bash
 $ redis:docker run —name redis -d  -p 6379:6379 -v /home/adventure/redis:/data redis /data/redis.conf
```

### 四：nodejs容器
   本小结介绍nodejs容器的安装。

* a:下载nodejs镜像

``` bash
  $ docker pull readytalk/nodejs
```

我这里下的是readytalk的nodejs镜像，当然也可以下载其他的nodejs镜像

* b:新建本地的nodejs项目

    + 1：首先在本地linux中新建一个nodejs项目，1:`express -e myapp`;2:进入myapp目录安装项目依赖，`npm install`。

    + 2：新建完nodejs项目之后，我们需要做一点小小的修改。进入routers目录下打开index.js,添加redis模块的依赖，

    ``` nodejs
    var redis=require("redis");
    var client=redis.createClient('6379','redis');
    ```
    这里要注意为了等会连接redis容器，创建redis client的时候’redis’不能写成“127.0.0.1”。
 
    + 3：修改router中的回调函数。

    ``` nodejs
    router.get('/', function(req, res, next) {
         client.incr('counter',function(err,counter){
          if(err){
              return next(err);
          }
          res.render('index',{title:'viewed'+counter+'times'});
      });
    }); 
    ```

    + 4：保存推出，安装redis依赖模块，`npm install redis`。

* c:启动nodejs容器并和redis容器相连

``` bash  
  $ docker run -d -p 3000:8080 --name myapp --link redis:redis -v /home/adventure/myapp:/data -w /data readytalk/nodejs npm start
```

其中-w表示nodejs的工作目录，—-link参数用来将本容器和其他容器相连。第一个redis表示要连接的容器名称，必须要和创建client（`var client=redis.createClient('6379','redis');
`）中的别名相同，第二个redis表示这个连接的别名。由于要做负载均衡，我这里创建2个nodejs容器，myapp和myapp1.

### 五：nginx容器
* a：下载nginx镜像

``` bash   
  $ docker pull nginx 
```

* b：修改本地nginx配置文件
   打开nginx.cof,在http下，添加upstream节点

``` vim   
upstream node-app {
      server myapp:3000;
      server myapp1:3001;
}，
```
然后将server节点下的location节点中的proxy_pass配置为：http:// + upstream名称，即“http://node-app”.保存并推出

* c:启动nginx

```bash
 $ docker run --rm --name nginx -p 80:80 -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf --link myapp:myapp --link app1:app1 nginx
```
用本地的conf替换docker中的conf配置。

### 六: docker常用命令

* 1：查看docker信息

``` bash
#查看docker版本
$ docker version
#查看docker系统信息
$ docker info 
```

* 2：对image操作

``` bash
#下载image
$ docker pull image_name
 #列出docker列表
$ docker images
```

* 3:启动容器
 
``` bash
#运行容器
$ docker run image_name
#交互式进入容器中 
$ docker run -i -t image_name /bin/bash
```

* 4：查看容器

``` bash
#列出当前所有正在运行的container
$ docker ps
#列出所有的container
$ docker ps -a
#列出最近一次启动的container
$ docker ps -l
```

* 5：保存对容器的修改
 
``` bash
# 保存对容器的修改; -a, --author="" Author; -m, --message="" Commit message
$ docker commit ID new_image_name
```

* 6：对容器的操作

``` bash
#删除所有的容器
$ docker rm ‘docer ps -a -q’
#删除单个容器
$ docker rm container_name/container_ID
#连接另一个容器
$ docker run  - - link container_name 
#后台运行
$ docker run -d
#每次运行完之后删除容器
$ docker run - -rm
#重启启动停止启动杀死一个容器
$ docker restart Name/ID
$ docker stop Name/ID
$ docker start Name/ID
$ docker kill Name/ID

#显示一个运行的容器里面的进程信息
$ docker top Name/ID

#从一个容器中取日志
$ docker logs Name/ID

#从容器里面拷贝文件/目录到本地路径
$ docker cp Name:/container_path  local_path
$ docker cp ID:/container_path local_path

#从本地拷贝到容器
$ docker cp local_path Name:/container_path
```

