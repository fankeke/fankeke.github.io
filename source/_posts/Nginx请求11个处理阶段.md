---
title: Nginx请求11个处理阶段
date: 2019-09-30 20:59:48
tags:
---

## 模块钩子嵌入

在处理请求时，Nginx是分为11个不同的阶段来完成的。在Nginx中，模块对请求进行操作的唯一途径是在这11个阶段中嵌入自己的钩子函数。

## 数据结构支持

### 嵌入范例

HTTP类型的模块嵌入钩子的过程是在解析配置文件的过程中完成的。通常会在模块上下文的ngx_http_post_config的地方插入自己的钩子，如ngx_http_access_module这个模块：


![](/images/Nginx请求11个处理阶/1.png)


![](/images/Nginx请求11个处理阶/2.png)


这个模块就在ACCESS阶段嵌入了一个自己的钩子:ngx_http_access_handler. （注意：也可以有模块在不同的阶段嵌入多个钩子或者同一阶段嵌入多个钩子，这都是可以的）
这些钩子的存放位置就在cmcf->phases这个数组中。

![](/images/Nginx请求11个处理阶/3.png)


![](/images/Nginx请求11个处理阶/4.png)


可以看到，这个结构实际上是一个二维数组的形式，即不同阶段的钩子都是分开存放在不同的一维动态数组中的。



### 钩子布局

为了对HTTP的钩子嵌入有直观的认识，下面是一个常规配置中其钩子的情景图：


![](/images/Nginx请求11个处理阶/5.png)

上图显示了常规情况下的钩子布局情况：
1，一共分为了11个阶段，“理论上”请求的处理过程是严格按照这个顺序来执行的。
2，并不是每个阶段都必须要有钩子，如上面的几个阶段是没有嵌入钩子的
3，每个阶段理论上可以嵌入任意多的钩子数量
4，第三方模块能够嵌入的阶段有限：0，1，3，5，6，9，10。而其它阶段（2，4，7，8）的钩子是由HTTP框架来嵌入的。



## 运行时“变身”

### 一维钩子数组

上面的钩子布局是由配置文件直接解析后生成的，但在处理http请求时，并不是按照上面的二维钩子数组来处理的，而是将其变成了一维数组。即运行时是以一维数组的形式来调用各个钩子的。
首先在cmcf结构体中除了cmcf→phases数组（即上面的那个二维数组）外，还有一个结构体cmcf->pahse_engine也和钩子函数的布局有关：


![](/images/Nginx请求11个处理阶/6.png)

![](/images/Nginx请求11个处理阶/7.png)

![](/images/Nginx请求11个处理阶/8.png)


cf->phase_engine的handlers字段是一个一维数组，它里面的内容由cmcf→phases二维钩子数组转换而来，它的存放的元素类型为ngx_http_phase_handlers_t结构体。
对于每个在二维数组中的钩子，都会在这个一维的handlers数组中对应着ngx_http_phases_handlers结构体（也即每个钩子都会有check，handler和next字段对应）。
在将二维钩子数组转换为一维钩子数组之前，需要对这个结构体的三个字段做简单的描述：
check: 一个包裹函数，每个钩子都会有一个check，且同一个阶段的所有钩子其check都是一样的，最重要的是，nginx从不直接调用钩子，而是调用其check，然后由check来调用钩子。
handler: 包裹的钩子函数，也即上面的钩子。
next：代表的含义相当于index，一维钩子数组下标。next表示从当前钩子所处阶段的下一个阶段中的第一个钩子在这个一维钩子数组中的下标，常用来快速跳到下一个阶段。
如果从上面的二维钩子数组转换为一维钩子数组来看，情景图如下：



![](/images/Nginx请求11个处理阶/9.png)


二维钩子数组中，每个阶段的钩子都按顺序被放在了相邻的一维钩子数组中.
补充说明：
1，r寻找到正确的server块（即r->srv_conf的正确指向）是在是这十一个阶段之前（在处理头部ngx_http_process_request_header的ngx_http_find_virtual_server中。也即在十一个处理阶段的前面。）
2，r寻找大正确的location块（即r→loc_conf的正确指向）是在FIND_CONFIG阶段。
3，r→main_conf肯定是唯一的（：））
4，在POST_REWRITE阶段（该阶段不能挂接自己的钩子，只会执行check函数）的next为1，暗示如果进行了rewrite跳转，那么下一个阶段会跳到FIND_CONF阶段去再次找寻到正确的location块。（通常如果没有rewrite的话，那么即phase＋＋，会进入PRE_ACCESS阶段。）


### 请求处理过程


在请求r的结构体中有一个字段为phase_handler,其类型为整型，这个整型为被赋值为一维钩子数组中的下标，由它来决定了请求在各个阶段的执行顺序或者跳转顺序。


![](/images/Nginx请求11个处理阶/10.png)


前面说过，在处理请求时，并不是直接调用各个钩子，而是调用了每个钩子的包裹函数-check函数:



![](/images/Nginx请求11个处理阶/11.png)


上面这段代码就是钩子函数被调用的核心逻辑：
以r->phase_handler为下标，调用相应的check函数，具体的check函数实现逻辑决定了r→phase_handler的变化，以及check函数的返回值决定了是否将控制流程交付给事件处理模块
即如果某个check函数返回了 NGX_OK，那么http模块就将控制流交付给了事件处理模块。
而check函数的返回值又和具体的钩子返回值有关，所以为了能够了解请求的执行顺序或跳转顺序，需要知道check函数对r→phase_handler的影响以及各个check函数的返回值。


## 各阶段顺序详解

### check包裹函数


下图总结了各个阶段的不同check包裹函数，其中有些阶段共用了一种check函数：


![](/images/Nginx请求11个处理阶/12.png)


有三个不同的阶段（POST_READ,PREACCESS,LOG)共用了check函数:ngx_http_core_generic_phase. 两个不同阶段(SERVER_REWRTIE,REWRITE)共用了ngx_http_core_rewrite_phase.其余都是各自有check函数。
而开发者需要关注的check只有4个（因为只可以嵌入的7个阶段中）：



![](/images/Nginx请求11个处理阶/13.png)


下面小节会逐步介绍它们中实现的逻辑是如何影响钩子的执行顺序的。


#### ngx_http_core_generic_phase


有三个阶段都共用了此check方法，如果要在post_read,preaccess,log阶段嵌入自己的钩子，那么必须对这个check有了解。
1 首先会调用模块嵌入的钩子，即handler. (当然第三方模块实现的钩子函数必须是非阻塞的），根据handler的返回值，它会有4中不同的逻辑。
2 若handler返回NGX_OK, 意味着当前阶段以及执行完毕，那么需要跳转到下一阶段的第一个钩子，即将r→phase_handler赋值为next，即使该阶段还有其它钩子，那么也将忽略不执行。同时check方法返回NGX_AGAIN.（返回AGAIN是保留了HTTP框架的控制流的）
3 若handler返回NGX_DECLINED,则会执行下一个钩子（举例来说，如果当前阶段有多个钩子，那么会继续在当前阶段执行下一个钩子，若该阶段只有这一个钩子，那么会流转到下一个阶段执行钩子），它将r→phase_handler＋＋。同时check返回AGAIN(保留控制流权）
4 若handler返回NGX_DONE/NGX_AGAIN,那么表示该handler没有处理完，需要多次调度才能完成（例如遇到了阻塞条件或者超时），这时候需要将控制权交出去，且r→phase_handler保持不变以便epoll在此触发时会继续调用此钩子。这时候check返回OK，交付控制流权给事件模块。
5 若handler返回其它值，表示执行遇到错误，需要结束这个请求，调用ngx_http_finalize_request.
故实现自己的钩子时需要根据逻辑确定返回值，进而影响到check的动作。



#### ngx_http_core_rewrite_phase

两个重写URL的阶段（server_rewrite,rewrite）共用了这个check。其逻辑和generic很相似
1 若handler返回NGX_DECLINED,则会执行下一个钩子（举例来说，如果当前阶段有多个钩子，那么会继续在当前阶段执行下一个钩子，若该阶段只有这一个钩子，那么会流转到下一个阶段执行钩子），它将r→phase_handler＋＋。同时check返回AGAIN(保留控制流权）
2 若handler返回NGX_DONE,那么表示该handler没有处理完，需要多次调度才能完成（例如遇到了阻塞条件或者超时），这时候需要将控制权交出去，且r→phase_handler保持不变以便epoll在此触发时会继续调用此钩子。这时候check返回OK，交付控制流权给事件模块。
3 若handler返回其它值(除DECLINED和DONE的其它值），表示执行遇到错误，需要结束这个请求，调用ngx_http_finalize_request.

PS：和上一个check包裹不同的是，这个check不会试图跳转到下一个阶段，即handler没有机会返回NGX_OK而得到正确处理。原因是，NGINX认为在重写URL这个点上，所有模块的优先级都是一样的，不应该存在先被调用的钩子会将其它钩子的执行权限“剥夺”的逻辑。



#### ngx_http_core_access_phase

(暂无）



#### ngx_http_core_content_phase（实际上的最后一阶段)



该阶段有两个特点：
a 这是第三方模块最经常嵌入的阶段
b 嵌入这个阶段的方式有两种（其它阶段都只有唯一的一种）
该阶段的作用是真正处理请求内容。
1 实际上该阶段是请求处理的最后一个阶段（LOG阶段是在请求结束的时候被执行的），那么就不会有跳转到下一个阶段的逻辑
2 其余阶段均为对所有的请求都有作用，而在CONTENT阶段，应该有这样的逻辑：即只对匹配了某个location的请求进行处理，这是该阶段的第二种嵌入方式
其实现方式如下：
1 在该location下的ngx_http_core_loc_conf_t结构体中赋值clcf→handler.
2 在请求r的字段中r→content_handler 中，将其赋值为的clcf→handler.
3 包裹函数的处理逻辑：



![](/images/Nginx请求11个处理阶/14.png)


PS:
1 若clcf的handler被调用，则这一阶段的其它钩子将被忽略。
2 若content钩子返回非DECLINED，则意味着该请求被处理完成，结束。
3 由于该阶段是实际处理请求的最后一阶段，所以需要对下一个钩子是否存在做有效性检查。
4 一般在content阶段的钩子会构造响应头部和响应体，然后发送出去。（如常见的static_handler获取静态文件然后发送的module）
