nginx事件模型中的instance变量，实际上是为了处理使用epoll时，可能出现的所谓“stale event”，先看下man的解释。
 
man 7 epoll：
```cpp
/*
       If you use an event cache or store all the fd’s returned from epoll_wait(2), then make sure to provide a way to mark  its  clo-
       sure dynamically (ie- caused by a previous event’s processing). Suppose you receive 100 events from epoll_wait(2), and in event
       #47 a condition causes event #13 to be closed.  If you remove the structure and close() the fd for event #13, then  your  event
       cache might still say there are events waiting for that fd causing confusion.
       One solution for this is to call, during the processing of event 47, epoll_ctl(EPOLL_CTL_DEL) to delete fd 13 and close(), then
       mark its associated data structure as removed and link it to a cleanup list. If you find another event for fd 13 in your  batch
       processing, you will discover the fd had been previously removed and there will be no confusion.
*/
```

大体的意思是说，在一批同时汇报上来的事件中，各个事件的处理时，fd可能会彼此影响，epoll提供给了程序员足够宽松的条件，去做你可以做的任何事件，副作用就是你需要照顾和考虑的细节就更多。这样说可能比较笼统，结合nginx看看吧。
 
nginx ngx_event_t结构中的instance变量是处理stale event的核心，这里值得一提的是，connection连接池(实际是个数组)和event数组(分read和write)。他们都是在初始化时就固定下来，之后不会动态增加和释放，请求处理中只是简单的取出和放回。而且有个细节就是，假设connection数组的大小为n，那么read event数组和write event数组的数量同样是n，数量上一样，每个连接对应一个read和write event结构，在链接被回收的时候，他们也就不能使用了。
我们看看连接池取出和放回的动作：
先看放回，一个请求处理结束后，会通过ngx_free_connection将其持有的连接结构还回连接池，动作很简单：
```cpp
c->data = ngx_cycle->free_connections;
ngx_cycle->free_connections = c;
ngx_cycle->free_connection_n++;
```

```cpp
/*
这里要主要到的是，c结构体中并没有清空，各个成员值还在(除了fd被置为-1外)，那么新请求在从连接池里拿连接时，获得的结构都还是没用清空的垃圾数据，我们看取的时候的细节：
*/
 
// 此时的c含有没用的“垃圾”数据
c = ngx_cycle->free_connections;
......
// rev和wev也基本上“垃圾”数据，既然是垃圾，那么取他们还有什么用？其实还有点利用价值。。
rev = c->read;
wev = c->write;
 
// 现在才清空c，因为后面要用了，肯定不能有非数据存在，从这里我们也多少可以看得出，nginx
// 为什么在还的时候不清，我认为有两点：尽快还，让请求有连接可用；延迟清理，直到必须清理时。
// 总的来说还是为了效率。
ngx_memzero(c, sizeof(ngx_connection_t));
 
c->read = rev;
c->write = wev;
c->fd = s;
 
// 原有event中的instance变量
instance = rev->instance;
 
// 这里清空event结构
ngx_memzero(rev, sizeof(ngx_event_t));
ngx_memzero(wev, sizeof(ngx_event_t));
 
// 新的event中的instance在原来的基础上取反。意思是，该event被重用了。因为在请求处理完
// 之前，instance都不会被改动，而且当前的instance也会放到epoll的附加信息中，即请求中event
// 中的instance跟从epoll里得到的instance是相同的，不同则是异常了，需要处理。
rev->instance = !instance;
wev->instance = !instance;
```

现在我们实际的问题：ngx_epoll_process_events
```cpp
// 当前epoll上报的事件挨着处理，有先后。
    for (i = 0; i < events; i++) {
        c = event_list[i].data.ptr;
        
        // 难道epoll中附加的instance，这个instance是在刚获取连接池时已经设置的，一般是不会变化的。
        instance = (uintptr_t) c & 1;
        c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);
        
        // 处理可读事件
        rev = c->read;
        /*
          fd在当前处理时变成-1，意味着在之前的事件处理时，把当前请求关闭了，
          即close fd并且当前事件对应的连接已被还回连接池，此时该次事件就不应该处理了，作废掉。
          其次，如果fd > 0,那么是否本次事件就可以正常处理，就可以认为是一个合法的呢？答案是否定的。
          这里我们给出一个情景：
          当前的事件序列是： A ... B ... C ...
          其中A,B,C是本次epoll上报的其中一些事件，但是他们此时却相互牵扯：
          A事件是向客户端写的事件，B事件是新连接到来，C事件是A事件中请求建立的upstream连接，此时需要读源数据，
          然后A事件处理时，由于种种原因将C中upstream的连接关闭了(比如客户端关闭，此时需要同时关闭掉取源连接)，自然
          C事件中请求对应的连接也被还到连接池(注意，客户端连接与upstream连接使用同一连接池)，
          而B事件中的请求到来，获取连接池时，刚好拿到了之前C中upstream还回来的连接结构，当前需要处理C事件的时候，
          c->fd != -1，因为该连接被B事件拿去接收请求了，而rev->instance在B使用时，已经将其值取反了，所以此时C事件epoll中
          携带的instance就不等于rev->instance了，因此我们也就识别出该stale event，跳过不处理了。
         */
        if (c->fd == -1 || rev->instance != instance) {
 
            /*
             * the stale event from a file descriptor
             * that was just closed in this iteration
             */
 
            ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "epoll: stale event %p", c);
            continue;
        }
 
        /*
          我们看到在write事件处理时，没用相关的处理。事实上这里是有bug的，在比较新的nginx版本里才被修复。
          国内nginx大牛agent_zh，最早发现了这个bug，在nginx forum上有Igor和他就这一问题的讨论：
          http://forum.nginx.org/read.php?29,217919,218752         
         */
 
        ......
    }
```
 补充：
为什么简单的将instance取反，就可以有效的验证该事件是否是“stale event”？会不会出现这样的情况：
    事件序列：A ... B ...  B' ... C，其中A,B,C跟之前讨论的情形一样，我们现在很明确，B中获得A中释放的连接时，会将instance取反，这样在C中处理时，就可以发现rev->instance != instance，从而发现“stale event”。那么我们假设B中处理时，又将该connection释放，在B'中再次获得，同样经过instance取反，这时我们会发现，instance经过两次取反时，就跟原来一样了，这就不能通过fd == -1与rev->instance != instance的验证，因此会当做正常事件来处理，后果很严重！
    不知道看到这里的有没有跟我有同样的想法同学，其实这里面有些细节没有被抖出来，实际上，这里是不会有问题的，原因如下：
    新连接通过accept来获得，即函数ngx_event_accept。在这个函数中会ngx_get_connection，从而拿到一个连接，然后紧接着初始化这个连接，即调用ngx_http_init_connection，在这个函数中通常是会此次事件挂到post event链上去：
```cpp
if (ngx_use_accept_mutex) {
    ngx_post_event(rev, &ngx_posted_events);
    return;
}
```
然后继续accept或者处理其他事件。而一个进程可以进行accept，必然是拿到了进程间的accept锁。凡是进程拿到accept锁，那么它就要尽快的处理事务，并释放锁，以让其他进程可以accept，尽快处理的办法就是将epoll此次上报的事件，挂到响应的链表或队列上，等释放accept锁之后在自己慢慢处理。所以从epoll_wait返回到外层，才会对post的这些事件来做处理。在正式处理之前，每个新建的连接都有自己的connection，即B和B'肯定不会在connection上有任何搀和，在后续的处理中，对C的影响也只是由于B或B'从连接池中拿到了本应该属于C的connection，从而导致fd(被关闭)和instance出现异常(被复用)，所以现在看来，我们担心的那个问题是多虑了。
