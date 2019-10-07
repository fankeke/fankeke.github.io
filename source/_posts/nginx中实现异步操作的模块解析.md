---
title: nginx中实现异步操作的模块解析
date: 2019-10-07 14:46:23
tags:
---

```
//模仿ups的写法，向后端发起异步请求，回掉

//在http世界里的r
typedef struct {
    ngx_connection_t                    *c;
    ngx_chain_t                         *out;
    ngx_chain_t                         *inlast;
    ngx_chain_t                         *in;
    ngx_http_asyn_send_request_cb        cb;
    ngx_int_t                            max_chain;

    void                                *param;
    ngx_msec_t                           timedout;
    ngx_event_t                          clean_evt;
    ngx_flag_t                           destroyed;
} ngx_http_asyn_send_request_ctx_t ;

ngx_http_asyn_send_request_ctx_t* ngx_http_asyn_send_request(ngx_str_t *request,
        ngx_addr_t *addr,
        ngx_http_asyn_send_request_cb cb,
        ngx_msec_t timedout,
        void *param)
{
    if (addr == NULLL) {
        return NULL;
    }

    pool = ngx_create_pool(256, ngx_cycle->log);
    if (pool == NULL) {
        return NULL;
    }

    pc = (ngx_peer_connection_pt*)ngx_pcalloc(pool, sizeof(*pc));
    if (pc == NULL) {
        goto connect_clear;
    }

    pc->log_error = NGX_ERROR_ERR;
    pc->log = (ngx_log_t*)ngx_palloc(pool, sizeof(ngx_log_t));
    *pc->log = *ngx_cycle->log;

    pc->get = ngx_http_asyn_get_peer;
    pc->free = ngx_http_asyn_free_peer;

    pc->name = &addr->name;
    pc->socklen = addr->socklen;
    pc->sockaddr = (struct sockaddr*)ngx_palloc(pool, pc->socklen);
    if (pc->sockaddr == NULL || addr->sockaddr == NULL) {
        goto connect_clear;
    }
    ngx_memcpy(pc->sockaddr, addr->sockaddr, pc->socklen);

    rc = ngx_event_connect_peer(pc);
    c = pc->connection;

    if (rc == NGX_OK || rc == NGX_AGAIN) {
        if (ngx_http_probe_test_connect(pc->connection) != NGX_OK) {
            if (cb) {
                cb(param, 0, NULL);
            }
            goto connect_clear_2;
        }
        c->pool = pool;
    } else {
        if (cb) { 
            cb(param, 0, NULL);
        }
        goto connect_clear;
    }

    ev_ctx = (ngx_http_asyn_send_request_ctx_t*)ngx_pcalloc(pool, sizeof(*ev_ctx));
    timedout <= 0 ? (ev_ctx->timedout = 60000) : (ev_ctx->timedout = timedout);

    //类似于初始化request,
    //这里ev_ctx应该要换个名字，叫做asyn，结构体ngx_asyn_t
    ev_ctx->param = param;
    ev_ctx->cb = cb;
    ev_ctx->c = c;

    // 这里没有http的概念、把http世界里面的r当作这里的ev_ctx了，
    // 那么如何与r联系起来：ev_ctx->param = param(param其实就是r)
    c->data = ev_ctx;

    c->write->handler = ngx_http_asyn_write_handler;
    //c->write->data = c; //默认就是:在ngx_get_connection中可以看到
    c->read->handler = ngx_http_asyn_read_handler;
    //c->read->data = c;

    //可读事件感觉像是丢了，很可怕
    //后来仔细翻阅代码，发现是在connect(ngx_event_connect_peer)的时候，框架已经帮你加入了可读可写事件：
    //在connect中可以看到：第一次的可读可写事件不用自己手动加入，但后续的需要
    //因为每次触发后，都从epoll中拿出来了(timer也一样）

    b = (ngx_buf_t*)ngx_pcalloc(pool, sizeof(*b));
    rlen = requlst->len;
    b->pos = b->start = (u_char*)ngx_palloc(pool, rlen+1);
    *ngx_cpymem(b->pos, request->data, rlen) = '\0';
    b->last = b->end = b->pos + rlen;

    //比较在http世界里面的r，这里很多地方都类似
    ev_ctx->out = (ngx_chain_t*)ngx_pcalloc(pool, sizeof(ngx_chain_t));
    ev_ctx->out->buf = b;
    b->memroy = 1;

    //如何通过事件找到ev_ctx？ev->data指向c，c->data指向ev_ctx
   ngx_http_asyn_write_handler(c->write);
    if (ev_ctx->destroyed) {
        return NULL;
    }
    return ev_ctx;

connect_clear:
    if (pool) {
        ngx_destory_pool(pool);
    }
    return NULL;
}

ngx_int_t
ngx_http_asyn_parse_http_retcode(ngx_chain_t *in, ngx_http_status_t *status, 
        ngx_http_request_t *r)
{
    ngx_int_t                       rc;
    ngx_memzero(status, sizeof(*status));

    //如果是从异常情况来的，这里in应该是为空的，或者解析不ok
    while(in) {
        rc = ngx_http_parse_status_line(r, in->buf, status);
        if (rc == NGX_ERROR) {
            return rc;
        }
        //唯一正常情况
        if (rc == NGX_OK) {
            return rc;
        }
        in = in->next;
    }

    //异常情况
    return NGX_ERROR:
}

static void
ngx_http_asyn_send_notify(ngx_http_asyn_send_request_ctx_t *ev_ctx)
{
    if (ev_ctx->cb == NULL) {
        return;
    }

    ngx_int_t               rc;
    ngx_list_t             *headers;
    ngx_http_request_t     *r;
    ngx_http_status_t       status;
    ngx_http_request_t      mock_r;

    //注意，调用到这里，有可能是错误情况也会走到这里（close里面调用的)
    //那么就需要判断(如果在ev_ctx里面再加上一个标记符号来表示异常情况就更好了

    //前面都没有判断异常情况，那么只有在里面去判断了
    rc = ngx_http_asyn_parse_http_retcode(ev_ctx->in, &status, &mock_r);
    if (rc != NGX_OK) {                                                                                                                                                                            [209/382]
        ev_ctx->cb(ev_ctx->param, 0, NULL);
        return;
    }

    r = (ngx_http_request_t*)ev_ctx->param;
    headers = (ngx_list_t *)ngx_palloc(r->pool, sizeof(ngx_list_t));
    if (headers == NULL) {
        ev_ctx->cb(ev_ctx->param, 0, NULL);
        return;
    }

    //用0调用cb，都是异常情况
    if (ngx_list_init(headers, r->pool, 20, sizeof(ngx_table_elt))
            != NGX_OK) 
    {
        ev_ctx->cb(ev_ctx->param, 0, NULL);
        return;
    }

    //解析头部可以参考ngx_http_request.c里面的代码
    rc = ngx_http_asyn_parse_header_lin(ev_ctx, ev_ctx->in, headers, &mock_r);
    if (rc != NGX_OK) {
        ev_ctx->cb(ev_ctx->param, 0, NULL);
        return;
    }

    ev_ctx(ev_ctx->param, status.code, headers);
}

//非常类似于ngx_http_close_request(r)
void ngx_http_asyn_close_request(ngx_http_asyn_send_request_ctx_t *ev_ctx)
{
    ngx_connection_t        *c;

    //ev_ctx->destroyed 表示调用过一次close了
    if (ev_ctx == NULL || ev_ctx->destroyed) {
        return;
    }

    ev_ctx->destroyed = 1;

    //是不是非常类似于r->connection找到c
    c = ev_ctx->c;
    
    //在这里面进行数据的解析、处理(如果是正常结束的流程)
    ngx_http_asyn_send_notify(ev_ctx);

    //从定时器中将相关事件都拿出来
    if (c->write->timer_set) {
        ngx_del_timer(c->write);
    }

    if (c->read->timer_set) {
        ngx_del_timer(c->read);
    }
    ngx_event_t *ev = &ctx->clean_evt;
    /*一般ev->data都指向c*/
    ev->data = ctx;
    ev->handler = ngx_http_asyn_clean_handler;
    ev->log = ngx_cycle->log;

    ngx_post_event(ev, &ngx_posted_events);
}
/* 把close_request的代码写下
 *
 * static void 
 * ngx_http_clsoe_request(ngx_http_request_t *r, ngx_int_t rc)
 * {
 *     ngx_connection_t         *c;
 *     r = r->main;
 *     c = r->connection;
 *
 *     if (r->count == 0) {
 *      ngx_log_error();
 *     }
 *
 *      r->count--;
 *
 *      if (r->count || r->blocked) {
 *          return;
*       }
*
*       ngx_http_free_request(r, rc);
*       ngx_http_close_connection(c);
* }
*/

static void
ngx_http_asyn_send_header(ngx_http_asyn_send_request_ctx_t *ev_ctx)
{
    if (ev_ctx->out == NULL) {
        ngx_http_asyn_close_request(ev_ctx);
        return;
    }

    ev_ctx->out = ngx_linux_sendfile_chain(ev_ctx->c, ev_ctx->out, 0);
    if (ev_ctx->out == NGX_CHAIN_ERROR) {
        ngx_http_asyn_close_request(ev_ctx);
        return;
    }

    if (ev_ctx->out != NULL) {
        if (ngx_handler_write_event(ev_ctx->c->write, 0) != NGX_OK) {
            ngx_http_asyn_close_request(ev_ctx);
            return;
        }
        ngx_add_timer(ev_ctx->c->write, ev_ctx->timedout);
    } else {
        if (ev_ctx->c->write->error != 0) {
            ngx_http_asyn_close_request(ev_ctx);
        } else {
            //正确无误地将所有头部信息都发送完了,等待接受
            ngx_del_event(ev_ctx->write, NGX_WRITE_EVENT, 0);
        }
        return;
    }
}

//触发收数据事件,典型的epoll编程模型+timer模型
static void
ngx_http_asyn_read_handler(ngx_event_t *rev)
{
    ngx_connection_t                    *c;
    ngx_http_asyn_send_request_ctx_t    *ev_ctx;

    c = wev->data;
    ev_ctx = c->data;

    if (ev_ctx == NULL) {
        return;
    }

    ngx_chain_t                         *cl;
    ngx_buf_t                           *b;

    ssize_t                              sl;

    if (c->destroyed || ev_ctx->destroyed) {
        return;
    }
    
    //和其他事件处理一样，都是先判断是否有timer触发，然后移除timer
    if (rev->timedout) {
        c->timedout = 1;
        ngx_http_asyn_close_request(ev_ctx);
        return;
    }

    if (rev->timer_set) {
        ngx_del_timer(rev);
    }

    for (;;) {
        if (ev_ctx->inlast == NULL
                || ev_ctx->inlast->buf->last == ev_ctx->inlast->buf->end)
        {
            cl = ngx_alloc_chain_link(c->pool);
            if (cl == NULL) {
                ngx_http_asyn_close_request(ev_ctx);
                return;
            }
            cl->next = NULL;
            cl->buf = ngx_create_temp_buf(c->pool, 2048);
            ++ev_ctx->max_chain;
            if (ev_ctx->max_chain > 100) {
                ngx_http_asyn_close_request(ev_ctx);
                return;
            }
            if (cl->buf == NULL) {
                ngx_http_asyn_close_request(ev_ctx);
                return;
            }
            
            if (ctx->in == NULL) {
                ctx->in = cl;
            } else {
                ctx->inlast->next = cl;
            }
            ctx->inlast = cl;
        }
        b = ctx->inlast->buf;
        si = c->recv(c, b->last, b->end - b->last);

        //如果接受数据有误，结束
        if (si == NGX_ERROR || si == 0) {
            c->read->error = 1;
            ngx_http_asyn_close_request(ev_ctx);
            return;
        }

        //此时事件出完成,1 继续加入读事件到epoll，2 防止超时,加入timer中
        if (si == NGX_AGAIN) {
            if (ngx_handle_read_event(rev, 0) != NGX_OK) {
                ngx_http_asyn_close_request(ev_ctx);
            } else {
                ngx_add_timer(rev, ev_ctx->timedout);
            }
            return;
        }
        b->last += si;
    }

    //继续收数据，一直到超时（表示没有数据可接受了，这是正常的情况，然后进入close处理）
    //其实这里有点不太合适，不应该在close中进行数据的解析，应该在接受完后进行
    //不过放在close中解析也并不耽误这里的政策逻辑运行
}

static void
ngx_http_asyn_clean_handler(ngx_event_t *ev)
{
    ngx_http_asyn_send_request_ctx_t        *ev_ctx;
    ngx_connection_t                        *c;
    ngx_pool_t                              *pool;

    ev_ctx = ev->data;
    c = ev_ctx->c;
    pool = c->pool;

    //在关闭c时，由于c的pool不是预先分配的，需要先将pool隔离开
    //单独是否，然后将c放入池子。直接放入池子会造成泄漏    c->pool = NULL;
    ngx_close_connection(c);
    if (pool) {
        ngx_destory_pool(pool);
    }
}

static void
ngx_http_asyn_write_handler(ngx_event_t *wev)
{
    ngx_connection_t                    *c;
    ngx_http_asyn_send_request_ctx_t    *ev_ctx;

    c = wev->data;
    ev_ctx = c->data;

    //事件可能会被多次调用，在调用之前，首先判断下事件的有效性
    if (c->destroyed || ev_ctx->destroyed) {
        return;
    }

    //可以看下官方代码：在做handler回掉时，一般都要先
    //判断下该事件是否是timer过期触发的，如果超时，则需要采取相应动作
    //比如关闭连接等(在ngx_http_request.c中也有类似的写法)
    //
    if (wev->timedout) {
        c->timedout = 1;
        //这里有点类似与http里面的ngx_http_close_request(r);
        ngx_http_asyn_close_request(ev_ctx); 
        return;
    }

    //如果事件还在timer树中，此时需要拿出来不让其再次由定时器触发
    //一般都这么使用：先判断超时，在拿出来
    if (wev->timer_set) {
        ngx_del_timer(wev);
    }

    ngx_http_asyn_send_header(ev_ctx);
}

```
