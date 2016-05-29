---
title: webbench
date: 2016-05-29 22:54:53
tags: [webbench,websocket bench,nodejs]
---

本文主要介绍两款网站压力测试工具，webbench和websocket bench，并且测试单机nodejs和socket.io的性能。<!--more-->webbench作为一款有名的网站压力测试工具，其主要特点是使用简单，功能较为丰富。其主要测试内容为每秒钟的请求数和每秒钟传输数据量。webbench不仅具有测试静态页面的能力，也能对动态页面进行测试，本文采用webbench测试单机nodejs的性能。websocket bench是一款能够用来测试websocket服务性能的工具，暂时支持Socket.IO，Engine.IO，Faye，Primus，WAMP等框架的测试。Socket.io作为一个实现websocket协议的nodejs框架，因为实现了服务器和浏览器的常连接，因此测试较为困难，本位采用websocket bench对其进行性能测试。

### 一：webbench的安装
webbench的安装需要依赖ctags,因此在安装webbench之前需要先安装ctags。

``` bahs
 sudo apt-get install ctags
```

安装好ctags之后，下载安装包并解压编译

``` bash
wget http://blog.s135.com/soft/linux/webbench/webbench-1.5.tar.gz
tar zxvf webbench-1.5.tar.gz
cd webbench-1.5
make && make install
```

### 二：webbench的使用
webbench使用非常简单，可以输入help查看命令。
``` bash
 webbench --help
 webbench [option]... URL
  -f|--force               Don't wait for reply from server.
  -r|--reload              Send reload request - Pragma: no-cache.
  -t|--time <sec>          Run benchmark for <sec> seconds. Default 30.
  -p|--proxy <server:port> Use proxy server for request.
  -c|--clients <n>         Run <n> HTTP clients at once. Default one.
  -9|--http09              Use HTTP/0.9 style requests.
  -1|--http10              Use HTTP/1.0 protocol.
  -2|--http11              Use HTTP/1.1 protocol.
  --get                    Use GET request method.
  --head                   Use HEAD request method.
  --options                Use OPTIONS request method.
  --trace                  Use TRACE request method.
  -?|-h|--help             This information.
  -V|--version             Display program version.
```
常用参数说明，-c表示客户端数量，-t表示时间。
``` bash
 webbench -c 500  -t  30   http://localhost:8080
```
### 三：Linux文件句柄数量受限制
默认情况下，linux用户能够打开的文件句柄数为1024,可以通过ulimit命令进行查看。
``` bash
ulimit -n
1024
```
也就是说在默认情况下，基于Linux的通讯最多允许同时1024个tcp并发连接。想要获得更高的tcp并发连接数，就必须修改Linux对用户进程可同时打开的文件数量的软限制和硬限制。其中
* 软限制是指linxu在当前系统能够承受的范围内进一步限制用户同时打开文件的数量
* 硬限制是指根据系统硬件资源状况计算出来的系统最多能够同时打开的文件数量。
修改单一进程能够同时打开文件句柄数主要有两种方法：

* 1,直接使用ulimit命令
``` bash
ulimit -n 4096
```
但是该方法设置的值只能在当前终端生效，并且设置的值不能高于硬连接数。

* 2,修改/etc/security/limits.conf文件，添加或者修改
``` bash
* soft nofile 1048576
* hard nofile 1048576
```
其中*表示对所有用户有效，soft表示软限制，hard表示硬限制，超过这个阈值，将会报错。nofile表示文件打开的最大数量。1028576是每个进程打开的文件句柄数，超过这个数需要重新编译Linux内核。

### 四：webbench测试结果
本文采用express框架实现了一个简单的nodejs项目，并采用webbench测试其主页性能。本文测试环境为：ubutun 14.04，4G内存，酷睿双核，内网环境下。
以下分别为100，1000，3500，10000和20000客户端下的性能。
![docker](../../../../img/100-c.png)
![docker](../../../../img/1000-c.png)
![docker](../../../../img/3500-c.png)
![docker](../../../../img/10000-c.png)
![docker](../../../../img/20000-c.png)
从以上测试结果来看，在单机情况下，当客户端连接数量在3500时，服务较为稳定，当客户端连接数达到10000时，服务开始变的不稳定，当连接数达到20000时，服务基本不可用。在实验过程中发现，当连接数达到几K时，服务的cup直线飙升达到90%以上，但是内存相对变化不是很大。

### 五：websocket bench安装
直接使用以下命令安装：
``` bash
 npm install -g websocket-bench
```
### 六：websocket bench的使用

```bash
websocket-bench --help
Usage: websocket-bench [options] <server>

Options:

  -h, --help               Output usage information
  -V, --version            Output the version number
  -a, --amount <n>         Total number of persistent connection, Default to 100
  -c, --concurency <n>     Concurent connection per second, Default to 20
  -w, --worker <n>         Number of worker(s)
  -g, --generator <file>   Js file for generate message or special event
  -m, --message <n>        Number of message for a client. Default to 0
  -o, --output <output>    Output file
  -t, --type <type>        Type of websocket server to bench(socket.io, engine.io, faye, primus, wamp). Default to socket.io
  -p, --transport <type>   Type of transport to websocket(engine.io, websockets, browserchannel, sockjs, socket.io). Default to websockets (Just for Primus)
  -k, --keep-alive         Keep alive connection
  -v, --verbose            Verbose Logging  
```
常用参数说明-a表示总的连接数，-c表示每秒并发连接数，-g可以指定特定的generate文件，-m 表示每个客户端的消息个数。
``` bash
 websocket-bench -a 100   -c 20   -g websocketBench.js  -m 1  -w 2 http://localhost:8080
```
其中的websocketBench.js格式如下：
``` vim
module.exports = {
       /**
        * Before connection (optional, just for faye)
        * @param {client} client connection
        */
       beforeConnect : function(client) {
         // Example:
         // client.setHeader('Authorization', 'OAuth abcd-1234');
         // client.disable('websocket');
       },

       /**
        * On client connectioonlinen (required)
        * @param {client} client connection
        * @param {done} callback function(err) {}
        */
       onConnect : function(client, done) {
         // Faye client
         // client.subscribe('/channel', function(message) { });

         // Socket.io client
         client.emit('online', { userName: client.id });

         // Primus client
         // client.write('Sailing the seas of cheese');

         // WAMP session
         // client.subscribe('com.myapp.hello').then(function(args) { });

         done();
       },

       /**
        * Send a message (required)
        * @param {client} client connection
        * @param {done} callback function(err) {}
        */
       sendMessage : function(client, done) {
         // Example:
          client.emit('chat', {from:client.id,to:'all',msg:'hello'});
         // client.publish('/test', { hello: 'world' });
         // client.call('com.myapp.add2', [2, 3]).then(function (res) { });
         done();
       },

       /**
        * WAMP connection options
        */
       options : {
         // realm: 'chat'
       }
    };
```

### 七：测试结果
本文采用socket.io实现了一个简单的聊天室，每一个登录的人可以对所有的人说话。测试环境为：ubutun 14.04，4G内存，单核，服务和测试工具都运行在同一台vitualbox虚拟机中。
![docker](../../../../img/chat.png)
![docker](../../../../img/100-a.png)
![docker](../../../../img/1000-a.png)
![docker](../../../../img/2000-a.png)
经过多次测试得知，在广播的情况下，服务端最多能够接受1000个左右的连接，未广播的情况下，服务端大概能接受2000个左右的连接。上述数据只在特定的情况下测试，仅供参考。

