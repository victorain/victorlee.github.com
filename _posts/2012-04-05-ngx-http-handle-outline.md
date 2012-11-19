---
layout: post
title: ngx 处理 http 请求的概要流程
category: nginx
original: true
tags: nginx 
---

本文介绍 ngx 处理 http 请求的概要流程。将介绍从 http 请求到来，到建立连接，
读取请求，开始处理请求这一过程发生了些什么以及这一过程涉及的重要的函数等。
希望借此能够对 ngx 处理 http 请求的过程有个提纲式的了解，以免阅读代码的过
程中因太多细枝末叶而陷入一头雾水找不着北的状态。


一些前提条件：   

- 使用的 ngx 代码为 1.0.10 版本。  
- 使用 epoll 事件模型。ngx 在 linux 环境在默认是这么干的。
- 使用 accept 互斥体。ngx 使用他来避免惊群。

先来一个概览图
<img width="100%" align="left" title="新窗口打开图片可以看得更清楚哦！" 
src="/images/postimg/handle-http-request-outline-1.png"/>

图中的”accept 事件“实际上指的是监听套接字的可读事件，因为监听套接字可读
的时候就表明有客户端的连接过来了。“其他io事件”指的是accept 事件以外的其
他 epoll 事件。

下面首先谈一下 ngx 事件循环的工作流程，然后在此基础上谈一下 ngx 对 http 
请求的处理流程。

### ngx 事件循环的工作流程

事件循环的核心函数是 ngx_process_events_and_timers 。这个函数主要干了四件
事情:抢占 accept mutex，等待并分拣事件，处理 accept 事件，处理其他io事件。
下面详细看一下：
  
- 抢占互斥体 ngx_accept_mutex
    
谁跟谁抢呢？是 ngx 的那些 worker 进程之间在抢。ngx_trylock_accept_mutex 
函数是抢互斥体的。这是个非阻塞操作，无论是否抢到都会立刻返回。如果抢占成功
那么会将全局变量 ngx_accept_mutex_held 设置为1，并将所有的监听套接字加入到
 epoll 中；否则，ngx_accept_mutex_held 设置为0，同时将所有的监听套接字从 epoll
 中删除。可以通过 ngx_accept_mutex_held 来判断 worker 进程是否抢到了 accept 
 互斥体。ngx_process_events_and_timers 函数调用 ngx_trylock_accept_mutex 之后，
如果抢占成功，则 flags 变量的 NGX_POST_EVENTS 位被设置为1，具体作用会下文
介绍。

- 等待事件，并分拣事件   

(void) ngx_process_events(cycle, timer, flags); 这个“函数”调用负责等待分拣
事件。实际上ngx_process_events 是一个宏，使用 epoll 事件模型时，对应于 epoll 
模块的 ngx_epoll_process_events 函数。ngx_epoll_process_events 函数中会调用 
epoll_wait 等待事件发生。如果 flags 参数的 NGX_POST_EVENTS 位没有被设置，则
直接调用事件的handler。如果 flags 参数被设置了 NGX_POST_EVENTS 位则会分拣事件，
对可读事件的分拣如下：如果是 accept 事件，那么放到 ngx_posted_accept_events
 队列，否则放到 ngx_posted_events 队列；对可写事件一律分拣到 ngx_posted_events 
 队列。
相关代码如下，这里仅列出读事件相关的：  

{% highlight c++ %}
events = epoll_wait(ep, event_list, (int) nevents, timer);  
......

if (flags & NGX_POST_EVENTS) {
    queue = (ngx_event_t **) (rev->accept ?
                   &ngx_posted_accept_events : &ngx_posted_events);

    ngx_locked_post_event(rev, queue);

} else {
    rev->handler(rev);
}
{% endhighlight %}


- 处理 accept 事件 和 处理其他io事件
  
首先处理 accept 事件，如下：
{% highlight c++ %}
if (ngx_posted_accept_events) {
    ngx_event_process_posted(cycle, &ngx_posted_accept_events);
}
{% endhighlight %}

然后后再处理其他 io 事件，如下：
{% highlight c++ %}
if (ngx_posted_events) {
    if (ngx_threaded) {
        ngx_wakeup_worker_thread(cycle);

    } else {
        ngx_event_process_posted(cycle, &ngx_posted_events);
    }
}
{% endhighlight %}

可以看到处理队列中的事件，都是调用了 ngx_event_process_posted，他所做的工
作就是挨个从队列中删除事件并调用其 handler 函数。accept 事件的 handler 是
 ngx_event_accept 函数。还记得吗？在初始化事件驱动时 ngx_event_process_init
  函数将 accept 事件的 handler 设置为了 ngx_event_accept 。 

### ngx 对 http 请求的处理流程

ngx 对 http 请求的处理流程，涉及到 http 请求到来时，如何建立连接，连接建
立后 ngx是如何从客户端连接上的读取数据，以及对 http 请求的处理是如何开始
的等内容。

#### 建立连接

当有客户端连接到来时，事件循环将 accept 事件分拣到 ngx_posted_accept_events 
队列，之后处理 accept 事件时，对应的 handler 函数 ngx_event_accept 被调用。
ngx_event_accept 主要做了以下事情：

- accept 客户端的连接
{% highlight c++ %}
ngx_socket_t       s;
...
s = accept(lc->fd, (struct sockaddr *) sa, &socklen);
{% endhighlight %}
- 为客户端套接字分配连接结构体及连接结构体的初始化

{% highlight c++ %}
c = ngx_get_connection(s, ev->log);
...
c->sockaddr = ngx_palloc(c->pool, socklen);
...
ngx_memcpy(c->sockaddr, sa, socklen);
...
// 连接结构体与监听套接字关联
c->listening = ls;
c->local_sockaddr = ls->sockaddr;
...
{% endhighlight %}

- 调用监听套接字的 handler,处理客户端套接字的可读事件  

ngx_event_accept 末尾调用了监听套件字的 handler 函数 ngx_http_init_connection ，
相关代码:
{% highlight c++ %}
ls->handler(c);
{% endhighlight %}

还记得吗？在 ngx 启动初始化时，ngx_http_add_listening 函数设置的监听套接字
的 handler 。下面来看看 ngx_http_init_connection 对客户端套接字的可读事件做
了什么处理。相关代码如下： 
 
{% highlight c++ %}
ngx_event_t         *rev;
...
rev = c->read;
rev->handler = ngx_http_init_request;
...
if (rev->ready) {
    /* the deferred accept(), rtsig, aio, iocp */

    if (ngx_use_accept_mutex) {
        ngx_post_event(rev, &ngx_posted_events);
        return;
    }

    ngx_http_init_request(rev);
    return;
}

ngx_add_timer(rev, c->listening->post_accept_timeout);

if (ngx_handle_read_event(rev, 0) != NGX_OK) {
    ngx_http_close_connection(c);
    return;
}
{% endhighlight %}

我们看到首先会将 客户端套接字的可读事件的 handler 设置为 ngx_http_init_request,
然后 如果 rev->ready 为真，则可读事件添加到 ngx_posted_events 队列(本文
只关注使用了 accept 互斥体的情况)，这样事件循环中接下来处理延迟事件时 
ngx_http_init_request 将会被调用。如果 rev 事件不 ready 则，添加客户端套
接字的可读事件到 epoll 队列，这样 ngx_http_init_request 要到下次被事件循
环 epoll_wait 到并分拣到 ngx_posted_events 队列，处理 ngx_posted_events 
队列时，才能被调用。  

跟踪代码发现，ngx_http_init_connection 函数中 rev->ready 总是假，什么时候
会为真呢？
下面这段代码是在 ngx_event_accept 函数中，为客户端套接字的 ngx_connection_t 
结构体初始化时，对可读事件的 ready 字段的设置。
{% highlight c++ %}
if (ngx_event_flags & (NGX_USE_AIO_EVENT|NGX_USE_RTSIG_EVENT)) {
    /* rtsig, aio, iocp */
    rev->ready = 1;
}

if (ev->deferred_accept) {
    rev->ready = 1;
#if (NGX_HAVE_KQUEUE)
    rev->available = 1;
#endif
}
{% endhighlight %}
我们看到当使用了 NGX_USE_AIO_EVENT 或 NGX_USE_RTSIG_EVENT 时会设置为1，
使用 epoll 事件模型时，所以不会在这里设置 ready字段。事件的 deferred_accept 
字段为真时，会设置 ready 为1，但是 deferred_accept 是个嘛呢，作为一个 todo 
研究吧。

#### 读取和处理请求

读取 http 请求开始于对客户端套接字可读事件的处理，上文已经提到在 ngx_http_init_connection
函数中将客户端套接字可读事件的 handler 设置为了ngx_http_init_request，
这就是处理 http请求的起始点了。来看看怎么处理 http 请求的

- ngx_http_init_request 函数

创建并初始化著名的 ngx_http_request_t         *r;

下面的代码来自于 ngx_http_init_request ，为了说明上的方便只摘出了重要
的语句，且个别语句的位置略有调整。

{% highlight c++ %}
ngx_connection_t           *c;
ngx_http_request_t         *r;
ngx_http_connection_t      *hc;

c = rev->data;
hc = ngx_pcalloc(c->pool, sizeof(ngx_http_connection_t));
r = ngx_pcalloc(c->pool, sizeof(ngx_http_request_t));
// hc 与 r 关联起来
hc->request = r;
r->http_connection = hc;
// c 与 r 关联起来，从现在开始 c 的data就是著名的 ngx_http_request_t  *r 了
c->data = r;
r->connection = c;
//设置 读事件的 handler 为 ngx_http_process_request_line
// 并调用事件的 handler
rev->handler = ngx_http_process_request_line;
r->read_event_handler = ngx_http_block_reading;
...
rev->handler(rev);
{% endhighlight %}

- ngx_http_process_request_line 函数

调用 ngx_http_read_request_header 和 ngx_http_parse_request_line，分别
负责读取请求和解析请求。请求被读取到 r->header_in 中。解析完成之后会调
用 ngx_http_process_request_headers
{% highlight c++ %}
rev->handler = ngx_http_process_request_headers;
ngx_http_process_request_headers(rev);
{% endhighlight %}

ngx_http_read_request_header 读取 http 请求时，如果 事件的 ready 标志为
真则直接读取数据，否则，调用 ngx_handle_read_event 添加可读事件到 epoll，
如下：

{% highlight c++ %}
if (rev->ready) {
    n = c->recv(c, r->header_in->last,
                r->header_in->end - r->header_in->last);
} else {
    n = NGX_AGAIN;
}
if (n == NGX_AGAIN) {
    if (!rev->timer_set) {
        cscf = ngx_http_get_module_srv_conf(r, ngx_http_core_module);
        ngx_add_timer(rev, cscf->client_header_timeout);
    }
    if (ngx_handle_read_event(rev, 0) != NGX_OK) {
        ngx_http_close_request(r, NGX_HTTP_INTERNAL_SERVER_ERROR);
        return NGX_ERROR;
    }
    return NGX_AGAIN;
}
{% endhighlight %}

- ngx_http_process_request_headers 函数

调用 ngx_http_process_request_header 解析请求的 header 部分，然后调用
 ngx_http_process_request 开始处理 request 。

- ngx_http_process_request 函数

通过调用 ngx_http_handler 函数来做处理 http 请求。ngx_http_handler 调用
了 ngx_http_core_run_phases , ngx_http_core_run_phases 开启了著名的 n 个
 phases 的处理，如下：
{% highlight c++ %}
ngx_http_phase_handler_t   *ph;
while (ph[r->phase_handler].checker) {
    rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);
    if (rc == NGX_OK) {
        return;
    }
}
{% endhighlight %}

关于 ”著名的 n 个 phases 的处理“ 会另有博文介绍。



ngx 处理 http 请求的概要流程就介绍到这里，本文完。

