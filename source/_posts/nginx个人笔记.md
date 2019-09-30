---
title: nginx个人笔记
date: 2019-09-30 21:32:10
tags:
---

### ls->handler钩子

ls结构体代表的是一个监听结构体，它的handler成员会随着nginx在哪一层来解释tcp流有所不不同


* http流
如果作为http来解释tcp流的话，其ls->handler为ngx_http_init_connection
(注：ls->handler的设置在解析http{}块的最后几步ngx_http_optimize_servers中设置的回调，包括创建ls结构体）.
 每个listenfd都会有一个ls结构体，其ls->handler为其建立连接后初始化的钩子，即在accept后被调用，具体是在ngx_event_accept中.
 每个lisenfd也会有一个c，rev，wev结构体与之对应，而其对应的rev的handler:rev->handler被设置为ngx_event_accept.这个过程在ngx_event_process_init中可以看到.
 在epoll_wait的时候，由event_list[i].data.ptr指向了c，继而会找到rev、wev，然后待用rev->handler,这里就是ngx_event_accept了.
 在ngx_event_accept中只是传递了ev结构体，那么如何找到了ls呢？答案就是ev->data会指向c，而在ngx_event_process_init中，每个listenfd对应的c的c->listening都会指向ls结构体，这样就找到了ls.
 实际上，在ngx_event_accept中，新生产的c的c->listening也指向了被其accept的ls结构体.:)

* stream流
对于四层流而言，在解析stream块时，ngx_stream_optimize_servers将ls->handler设置为乐ls->handler=ngx_stream_init_connection.


### 事件handler


今天看了下框架层面的代码


* 模块的init_process是在worker中执行的，所以是每个worker执行，而init_module是master的时候执行的，注意区分。
* 在Nginx中，连接c才有读写事件，c->rev/wev,请求没有这个概念，我们说读事件／写事件，都是说在该连接上发生的事件，而请求只是在连接之上的抽象，它没有读写事件的资格
* 在ngx_conf_parse中发现，cycle是main中的一个局部变量，同时注意区分cycle,cycle->conf_ctx(就是那个著名的void**),conf (ngx_conf_t*,经常看到的cf）。
* 所有的事件钩子的参数都只有一个ev，而ev的data字段会指向其c字段，而c字段的data字段会指向r字段，这样就把c、r都找到了.（这里注意空闲的c的data字段是指向下一个空闲的c的）
* 注意区分事件handler和IO-handler的区别：事件处理handler是ev->handler,而IO-handler是存在c上的c->read/write，c->read=ngx_unix_recv,c->write=ngx_uint_send等等。而在ev->handler中可以使用它们来做IO。这中做法好棒，可扩展性强.


### 代码层面

* 最近写创建共享内存时候发现的,ngx_shared_memory_add,如果未赋值shm->init,shm->data,会导致-t检查不通过：


```
sudo ./nginx-1.6.2/obj/nginx -p `pwd` -c conf/nginx.conf -t
nginx: the configuration of file /home/ke/test/web_server/nginx/try/conf/nginx.conf syntax is ok
``` 
/*后面一句successfule居然就没有啦,加上后就可以了*/
shm_zone->init=ngx_http_xxx_init_zone;
shm_zone->data=shmctx;
```

* 添加一种和event、http同等级的模块，写完后，发现配置指令错误：

```
sudo ./nginx-1.6.2/obj/nginx -p `pwd` -c conf/nginx.conf -t
nginx:[emerg] "xxx" directive is not allowed here in /home/ke/test/web_server/nginx/try/conf/nginx.conf:17
nginx:configuration file /home/ke/test/web_server/nginx/try/conf/nginx.conf test failed
配置示例如下：
events {
        worker_connection 1024;
}
mylevel {
        xxx;
}
后来发现问题是block钩子没有解析完导致的，把ngx_xxx_block解析写完成即可
```


### cmd中的offset

以前在写cmd时候，之前以为offsetof(xxx,xx) 写了之后，就会对最后调用set函数时的conf有影响，即已经帮你找到了最终需要操作的字段。
现在经过验证，其实offsetof字段没有参与conf计算，你需要根据offsetof来自己确定最终的字段。如下：


![](/images/nginx个人笔记/1.png)

![](/images/nginx个人笔记/2.png)

  最后经过debug调试:

![](/images/nginx个人笔记/3.png)


打印出来的conf都是同一个位置，即loc_conf结构体，而并没有去在帮你找到具体的字段，需要你自己根据offsetof来找。parse_conf函数也证明了这点,即在拿到conf的时候并没有offset的参与:


![](/images/nginx个人笔记/4.png)


那么既然是这样，为何还要设置offsetof字段呢？反正可以通过conf来得到结构体，然后利用结构体就可以取得各个字段了。
其实，如果是自己写的set函数，这个offsetof是没有啥作用的，但是在采用预定义的那些set（如ngx_conf_set_flag_slot等），这时候就必须指定offsetof了，因为预定义的set使用了offsetof直接拿到具体的字段。


### content模块发送数据

在调用ngx_http_output_filter发送数据时，有时候会发现请求命令(如curl等客户端）迟迟不返回，这时候可用检查下buf的标志为buf->last_buf是否置位了。


### http模块的几个钩子的执行顺序

在阅读C模块时，老喜欢忘记ctx中的几个钩子执行顺序，这里做个记录：
|
create_main
|
create_srv
|
create_loc
|
preconfig – 一般在此处设置模块需要暴漏的变量
|
ngx_conf_parse – 解析http{},这样cmd中的钩子会全部执行
|
init_main
|
merge_srv
|
merge_loc
｜
init_locations –创建location的三叉树
|
init_phase – 创建phase表
|
post_config –一般这里会嵌入模块在各个阶段的钩子，或者filter钩子
|
optimze_server –初始化所有的http listenfd，包括挂接ls->handler

其实所有的流程都在ngx_http_block中一目了然，这里方便做个记录



### 阅读dyups模块的笔记

周末抽空阅读了下dyups早期的代码,这时候还只能在单worker中工作,所以以下分析主要是以这个版本的代码为基础的）

https://github.com/yzprofile/ngx_http_dyups_module/tree/0c169da7dceeeecf956a84aa25ba1dcd108ec98e
dyups模块与upstream模块结合得很紧密，主要是通过dyups暴漏出的几个API来操纵upstream模块里的数据结构体。关于获得操作比较易懂，通过获取upstream模块的数据然后展示出来，重点是DELETE和UPDATE的逻辑。



1 DELETE

* 删除一个upstream块的核心逻辑是：将upstream模块的某个ngx_htt_upstream_srv_conf_t结构体下的所有server的状态都置为down us[i].down=1，然后再次初始化调用了uscf->peer.init_upstream(默认情况下该钩子为ngx_http_upstream_init_round_robin).
(其实这里不太理解为啥要做这个调用，我实践了下，感觉不做的话也没啥影响) 这里需要注意的是，uscf->peer有两个与初始化相关的钩子 init_upstream和init:


* 前一个钩子是在upstream模块的init_main_conf中配置文件解析后被调用的，和dyups模块这里调用的效果一样，初始化upstream模块里的数据


* 后一个钩子init是在每个请求（那些需要xxx_pass的）将要转发时被调用的，之所以是在这里而不是在初始化阶段，在《Nginx开发从入门到精通》里面解释的很好：
“nginx收到一个请求后，发现如果要访问upstream，就会执行对应的peer.init函数，这是在初始化配置时设置的回调函数(在调用init_upstream也即ngx_http_upstream_init_round_robin中时被设置）。这个函数的作用时构造一张表，当前请求可以使用的upstream服务器被依次添加到这张表中。之所以需要这张表，最重要的原因是如果upstream服务器出现异常，不能提供服务时，可以从这张表中取得其它服务器进行重试操作，此外这张表也可以用于进行负载均衡的计算。之所以构造这张表的行为放在这里而不是在前面初始化配置阶段，时因为upstream需要为每个请求提供独立隔离的环境。”


* 这样就可以解释为何dyups的DELETE仅仅将其us[i].down置位就能到达“删除”某个upstream块的效果了，而每个请求在prox时调用init来构造可用节点表时,发现每个节点都down，所以这个请求就502了.


2 UPDATE(POST)

* 增添

