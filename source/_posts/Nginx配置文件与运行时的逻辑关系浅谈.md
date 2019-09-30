---
title: Nginx配置文件与运行时的逻辑关系浅谈
date: 2019-09-30 20:10:34
tags:
---

## 一 配置示例

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image1.png)

以上面的Nginx配置为例，本次讨论都基于这个配置文件。在配置中忽略了一些额外选项。只保留了http块，server块，location块。


## 二 配置文件解析过程

Nginx是在将配置文件解析和加载完之后，然后开始fork子进程，进行连接处理。所以对于明白Nginx是如何加载配置文件以及加载后发生了什么很有必要理解清楚。
在加载和解析配置文件时，主要的工作就是调用各个模块的（包括核心模块，http模块等内置模块，也包括第三方模块）钩子函数。
而每个不同类型的模块的关键不同之处在于它们的ctx结构体的不同，具体如下：


CORE类型:


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image2.png)

Core类型的模块的ctx结构只有两个钩子。在ngx_init_cycle中被调用。下面看其调用顺序（关键）

ctx->create_conf(cf) (目前只有两个Core类型的module注册
        ｜
        ｜
ngx_parse_conf(file) 这个函数是解析配置文件的核心函数，在里面会有一堆的模块的cmd结构体中的钩子被调用。

注意：当这个函数返回的时候就代表整个配置文件已经解析完了。
在解析到event子段，http子段的时候，有两个特殊的cmd钩子被调用：ngx_event_block,ngx_http_block.
而这两个钩子又会使得event类型模块中的ctx结构中的钩子，http类型模块中的ctx结构体中的钩子被一一调用。


### 1 遇到events子段被ngx_event_block激活的情况：

Event类型的ctx的结构体钩子：

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image3.png)


其中actions为一个钩子数组。其被调用的顺序：
module->create_conf
|
|
ngx_parse_conf(…) 进入 解析event{}部分，又一堆的event类型模块的cmd命令钩子被调用。
｜
｜
module->init_conf
至此，event{} 中的解析完成，ngx_event_block返回.



### 2 遇到http子段被ngx_http_block激活的情况（关键点）

http类型的ctx的结构体钩子:

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image4.png)


ctx->create_main_conf
ctx->create_srv_conf
ctx->create_loc_conf
ctx->preconfigurat
ngx_parse_conf() //进入解析http{}中的指令
在此中又会有一堆的cmd钩子函数被回调。其中包括解析遇到server指令的ngx_http_core_server(),解析upstream指令的ngx_http_upstream两个特殊钩子，
而在ngx_http_core_server钩子中:
1,会继续调用各个模块的create_srv_conf,create_loc_conf,
2,遇到location指令，会遇到一个特殊钩子:ngx_http_core_location，它会继续调用各个模块的create_loc_conf钩子 （所以，在http模块的ctx的钩子中，这些钩子不只是被调用一次，而是按照配置文件的格式被多次调用）
ctx->init_main_conf
ctx->merge_srv_conf
ctx->merge_loc_conf
ctx->postconfiguration;/// 这个钩子就是大部分模块获得拦截流量处理机会的关键，它一般会在phases数组中（非常重要，存放了http请求处理的11个阶段的各个钩子）插入自己的钩子，或者是在filter链表钩子中挂接自己的钩子,或者是设置clcf结构体的handler成员（后面详述）
ngx_http_init_phases_handler
ngx_http_optimize_servers (这两个函数也非常重要，不过不属于这里需要关注的：））

至此，ngx_http_block钩子就执行完毕，http{}中的指令也都解析完毕。配置文件就解析完成了

返回到在core类型中被调用的ngx_parse_file函数：调用 module->init_conf //目前只有一个core类型的模块注册了此钩子

这样整个配置文件就解析完毕了。其Nginx的基本数据结构的创建和初始化也都准备完毕。
从上面的分析来看，其钩子函数的执行符合 总－分－总 的调用架构。



## 三 解析后的布局

### 1 基本配置的布局

以上面的配置文件为例，观察http ｛｝块中的结构体布局：

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image5.png)

以上为以基本配置为范例解析后，nginx创建的有关http服务的结构布局。可以看到有多个srv块和loc块的冗余结构。
说明：
1，在http块中，存在完整的main，srv，loc三个上下文结构体数组，且main结构体数组是全局唯一的，比如ngx_http_core_main_conf,这些main_conf结构对于全局结构题的组织具有重要作用。
2，在srv块下面，它的main上下文结构体数组是继承的http快的，但srv块有着自己独立的srv和loc结构体数组
3，在loc块下面，它的main、srv上下文结构体数组都是继承自己上一层的srv的srv和loc结构体数组，但有着自己独立的loc结构体数组。


### 2增添一个模块后的布局图
 
下面增添一个三方模块，然后看其创建的结构体在整个布局图中的位置是如何安放的。模块的名字叫做ngx_http_test_access_module


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image6.png)

其ctx结构体如上，为该模块创建一个属于loc位置的结构体 ngx_http_test_access_loc_conf_t。


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image7.png)


那么当nginx加载模块并解析完配置文件后，其布局为：

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image8.png)

可以看到，仅仅一个模块创建的loc结构体，在当前的配置结构下就有6份冗余。（取决于配置的结构体，当前是一个http块，两个server块，三个location块）


## 四 配置指令与请求处理顺序的相互作用

### 1 请求的11个处理阶段
在ngx_http_main_conf_t结构体中（这个结构体全局仅一份），有种要的成员phases数组。

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image9.png)

它是相当于一个二维动态数组，其结构体如下图所示：

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image10.png)

其中每个阶段都有一系列的属于该阶段的钩子函数 。（注意，这只是所有被组册的钩子函数刚开始的组织结构，后期会将这个二维钩子数组转变为一维的钩子数组）


### 2 注册钩子

如上面所说的，http模块在组册钩子的时候会在ctx->postconfigruration函数。在test_access模块中我们注册自己的钩子：


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image11.png)

选择在access阶段组册钩子： ngx_http_test_access_handler.


### 3 配置指令与钩子间的执行关系

#### 1 指令出现与否的与钩子的关系

在配置文件中的指令配置、指令位置、指令配置与否这些条件在本质上都不能改变nginx的请求处理经过的顺序，而对请求的如何处理只和钩子函数的实现逻辑有关。
如上，因为在access阶段注册了钩子，那么所有的请求都会经过这个钩子的处理，即使不配置任何指令。
我们做个实验：在ngx_http_test_access_handler钩子中不做任何处理，直接返回403foridden。在模块的cmd指令做不设置任何指令，配置文件中不做任何配置。

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image12.png)

然后访问：

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image13.png)


可以看到，任何访问都将被foridden，即使配置文件里面没有配置任何test_access的指令。
而我们也可以配置一个指令，让其具有开关的功能，如果开关打开，则放行，否则forbidden。


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image14.png)

其配置文件更改为：

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image15.png)


在／下没有配置，默认为UNSET，在server1_test1下面打开，在server2_test1下面关闭。然后在钩子中实现这样的逻辑：


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image16.png)

即如果没有配置test_access选项或者选项开关关闭，则403，否则放行。然后验证其效果：


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image17.png)

可以看到，没有配置或者配置关闭的location，返回了403，而开关打开的server1_test1可以放行。

#### 2 指令出现的位置与钩子函数的关系

指令出现的位置与实际造成的效果是钩子函数中的逻辑与实现决定的。
在access模块中（内置的access模块，不是这里的第三方模块），有一种现象就是如果将access指令配置在http块，那么所有的location块都会受到这个access指令的影响效果。而如果只将access指令配置在location块下，则只会对当前的那个location块生效。
这样容易有一种错觉，好像指令的位置和它管辖的范围会有关系（尽管确实是这样：）），但我们得清楚它的实现机制。
以test_access模块为例，如果在http层配置后会不会使得全局的location都生效呢？ 如下配置

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image18.png)

将test_access 放在了http层，且打开开关。然后访问，其结果为：


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image19.png)


可以看到，放在http层的开关，并没有对任何一个location起到放行的作用。也就是说，这些talcf->passed要么没有被设置，要么被关闭了。
由上面的分析可以知道，在创建ngx_http_test_access_loc_conf_t结构体时候，一共创建了7份，而配置在http层的test_access指令，只是把http层的ngx_http_test_access_loc_conf_t的passed打开了。其余的ngx_http_test_access_loc_conf_t的passed仍旧是未打开状态。为了达到像access一样的“配置范围决定作用范围的效果“，需要将http层的ngx_http_test_access_loc_conf_t的passed的状态更新到各个ngx_http_test_access_loc_conf_t中，而ctx->merge_loc_conf钩子就是干这个工作的。


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image20.png)


其逻辑就是：如果子层的passed未配置（UNSET为－1），则将其上层的passed的值赋给子层的passed。（注意这行的红色字体，如果子层已经配置，则不会继承，当然，这一切都将由你自己写的merge钩子来决定）
然后再看访问结果：


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image21.png)

ok，全部都放行。

所以，从这里可以看到，并不是单纯的指令的位置而决定了其影响的范围，而是钩子函数的逻辑来决定的。



## 五 指令配置顺序与请求执行顺序

经常会有这样的感觉：指令配置的顺序决定了请求处理的顺序，如图所示的配置文件：

![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image22.png)

为了简单的做实验，这里直接用写了lua）
例如，上面配置了access_by_lua_file是影响整个server块的，在location/test下面配置了rewrite_by_lua_file。
其两个lua文件简单的输出标示：


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image23.png)

然后访问 servername2:8039/test
虽然access_by_lua_file指令的位置是配置在rewrite_by_lua_file指令位置之前的，但是这个请求被处理的顺序没有任何关系，而是严格按照其11个阶段的顺序来执行的。
由于rewrite阶段在access阶段之前，故有下面的结果：


![](/images/Nginx配置文件与运行时的逻辑关系浅谈/image24.png)

会先执行rewrite的lua文件。（由于ngx.say的实现是直接返回了的，所以这里没有继续执行access阶段的文件，这和ngx.say的实现有关，但并不影响这里讨论的话题）

所以，指令的配置顺序不能和请求的执行顺序相等。


## 六 rewrite指令详解
//待续

