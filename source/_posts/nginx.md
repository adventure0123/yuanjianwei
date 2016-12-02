---
title: nginx
date: 2016-11-29 22:18:38
tags: nginx
---

&emsp;&emsp;nginx作为一个著名的http和反向代理服务器已经得到了广泛的应用，本文主要简单介绍nginx的简单原理和配置。<!--more-->nginx在启动之后，默认会以daemon方式在后台运行，后台进程包括一个master进程和多个worker进程。master进程主要用来监控worker进程，包括：接受外界的信号，向各个worker发送信号，监控worker进程的运行状态。并且在worker进程异常退出之后，重新启动新的worker进程。而基本的网络请求都是在各个worker进程中处理的，每个worker进程之间都是对等的。一个请求只会在一个worker进程中处理，同时一个worker进程也不能处理其他进程的数据。一般为了提高效率和减少上下文切换的开销，我们将worker进程数量设置成和机器cup数量一致。nginx的进程模型，可以由如下图来表示：
<center>
![nginx](../../../../img/nginx.png)
</center>
nginx的高并发主要是因为其采用了异步非阻塞的方式来处理请求。如果采用阻塞调用，当读写事件还没有准备好的时候，阻塞调用就会进入内核等待阶段，cup将会等待直到读写事件处理完，这样会导致所有的请求都卡在了io上，cpu大量空闲没人使用。如果采用非阻塞，当读写事件还没有准备好的时候，会返回EAGAIN,告诉cpu事件还没有准好，让他先去处理其他的事情。过一会cpu又来检查一下读写事件是否好了没，虽然不阻塞了，但你不得不不时的来检查一下事件的状态，由此也带来了不小的开销。epoll和kqueue等提高了一种机制，可以让你同时监控多个事件，调用他们的是阻塞的，但是可以设置超时事件，在超时事件之内，如果事件准备好了就返回。以epoll为例，当事件没准备好时，放到epoll里面，当事件准备好了，我们就去读写，当读写返回EAGAIN时，我们再次把他放入epoll中。这样只有当事件准备好， 我们才去处理他，解决了前面说的两个问题。
&emsp;&emsp;在Ubuntu下，我们可以直接使用apt-get来安装nginx:
```vim
sudo apt-get install ngxin
```
安装之后可以直接启动nginx
```vim
sudo nginx
```
默认的nginx设置了转发80端口，我们可以在浏览器中输入localhost来进行访问。nginx的默认配置文件位于：/etc/nginx/nginx.conf
* 1 一些nginx.conf全局配置介绍：

+ a daemon on|off 默认为on
是否以守护进程的方式运行nginx,关闭守护进程可以方便我们调试nginx.

+ b master_process on|off 默认on
是否以master/worker方式进行工作，在默认情况下nginx是以一个master管理多个worker进程的方式来运行的，关闭后nginx就不会fork出多个进程，而是通过master来处理请求。

+ c work_processes number 默认为1
在master/worker运行方式下，worker进程的数量，一般情况下降worker的数量设置为cpu的核数。

+ d worker_limit_nofile 默认为操作系统的限制
该值为worker进程可以打开的最大文件描述符的数量。
ngxin启动也可以通过-c来指定配置文件的路径

* 2 nginx的重启，有时候我们修改了conf文件，想要在nginx不关闭的情况下平滑重启ngixn,可以使用reload命令。
```vim
nginx -s reload
```
在修改conf文件之后，需要先验证下配置文件是否正确，可以使用-t命令。
```vim
nginx -t -c conf文件
```
如果出现nginx.conf syntax is ok。
nginx.conf test is successful说明配置文件正确。
* 3 nginx停止可以使用
```vim
nginx -s stop #快速停止nginx
nginx -s quit #完整有序停止ngix
```
&emsp;&emsp;我们可以修改config里面的location属性来将请求用不用的逻辑处理。例如静态文件分发如下：
```vim
location ~ /(javascript|css|images) {
               root /usr/share/nginx/statics;
             }
```
这个表达式的意思是将请求中的js,cs,images请求都去root目录下获取。除了静态分发nginx还有一个重要的功能就是方向代理和负载均衡。配置起来也是非常简单，例如：
```vim
location / {
    proxy_pass localhost:8080;
}
```
这样所有的请求都被反向代理到本地的8080端口了。要想实现nginx就需要用到upstream模板了。
```vim
upstream backend {
    ip_hash;    
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com;
    server backend4.example.com;
}
location / {
    proxy_pass http://backend;
}
```
我们在upstream中指定了一组的机器，并且将这个组命名为backend,这样只要在proxypass中将请求转发到backend这个upstream中就可以在这4台机器中实现负载均衡了，并且负载均衡的策略是根据用户的ip进行hash。除了根据ip hash，nginx还有轮询，最少连接，加权等策略。最小连接如下：
```vim
upstream backend {
    least_conn;    
    server backend1.example.com;
    server backend2.example.com;
}
```
加权连接如下：
```vim
upstream backend { 
    server backend1.example.com weight=2 max_fails=3 fail_timeout=30s;
    server backend2.example.com  weight=1 max_fails=3 fail_timeout=30s;
}
```
在上面的例子中weight=2表示每收到3个请求，前两个请求会被分发到第一个服务器，第三个请求会被分发到第二个服务器。并且权重的负载均衡可以和基于ip地址hash的策略组合在一起使用。
