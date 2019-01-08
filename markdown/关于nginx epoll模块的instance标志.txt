man 7 epoll会发现这个东西,就是使用epoll中会遇到的问题：

`
o If using an event cache… If you use an event cache or store all the file descriptors returned from epoll_wait(2), then make sure to provide a way to mark its closure dynamically (i.e., caused by a previous event’s processing). Suppose you receive 100 events from epoll_wait(2), and in event #47 a condition causes event #13 to be closed. If you remove the structure and close(2) the file descriptor for event #13, then your event cache might still say there are events waiting for that file descriptor causing confusion.
`

这种事件也可以叫做stale event，而下面是man手册提出的解决方法：

`
One solution for this is to call, during the processing of event 47, epoll_ctl(EPOLL_CTL_DEL) to delete file descriptor 13 and close(2), then mark its associated data structure as removed and link it to a cleanup list. If you find another event for file descriptor 13 in your batch process‐ ing, you will discover the file descriptor had been previously removed and there will be no confusion.
`

问题很简单，由于大部分的服务器都会有一个连接池。而连接池是通过fd来进行定位，前面处理的事件会影响后面的事件，比如关闭掉了后面的事件，而后关闭掉的事件在当前的循环中还是会被处理，这种情况很好处理，比如设置fd为－1，就可以检测，可是还有一种情况，就是当你关闭了fd，然后设置－1之后，恰好接收到的新的连接的fd刚好和刚才close的fd的值是一样的。此时就会引起混乱了，也就是说我们需要区分事件是不是stale event，或者说是我们方才释放掉的fd被重新使用，而nginx中并没有按照上面man手册里面的方法，它的做法很巧妙，我们来看nginx如何做的。 首先要知道在nginx中是存在一个连接池的，所有的连接的获取和释放都是通过连接池来进行的，nginx中连接池很简单，就是一个简单的数组，有一个free_connections变量保存了所有可以使用的连接，它是一个链表，它的构造是这样子的，每个连接都有一个域data，如果释放一个连接，则这个连接的data就指向当前的free_connects,然后当前的释放的连接直接指向free_connections,也就是一个将连接加入free链表的 动作。
```cpp
void
ngx_free_connection(ngx_connection_t *c)
{
    /* ngx_mutex_lock */
    //保存当前的free_connections.
    c->data = ngx_cycle->free_connections;
  
    //将释放的连接加入到free connections
    ngx_cycle->free_connections = c;
  
    //可用连接数加1
    ngx_cycle->free_connection_n++;
      
    /* ngx_mutex_unlock */
    if (ngx_cycle->files) {
        ngx_cycle->files[c->fd] = NULL;
    }
}
```
然后我们来看如何得到一个连接，这里事件结构有个很关键的变量instance，它就是一个标记，用来标记两个连接的，因为nginx中刚刚释放的连接有可能会被马上使用的，因为free_connections它是一个类似与栈的东西，也就是在循环中有可能会遇到刚释放的连接又被使用（fd相同)，而我们此时并不知道，因此这里这个instance就是用来判断这个的。
```cpp
ngx_connection_t *
ngx_get_connection(ngx_socket_t s, ngx_log_t *log)
{  
	ngx_uint_t instance;
	ngx_event_t *rev, *wev;
	ngx_connection_t *c;

	...

	//可以看到获取到的是最新被释放的连接
	ngx_cycle->free_connections = c->data;
	ngx_cycle->free_connection_n--;

	/* ngx_mutex_unlock */
	if (ngx_cycle->files) { 
	    ngx_cycle->files[s] = c;
	}

	//保存对应的event，避免内存再次分配
	rev = c->read;
	wev = c->write;

	ngx_memzero(c, sizeof(ngx_connection_t));
	c->read = rev;
	c->write = wev;
	c->fd = s;
	c->log = log;

	//获取instance
	instance = rev->instance;
	ngx_memzero(rev, sizeof(ngx_event_t));
	ngx_memzero(wev, sizeof(ngx_event_t));

	//这里可以看到将instance去反，用以区分是否是刚才被释放的
	rev->instance = !instance;
	wev->instance = !instance;
	rev->index = NGX_INVALID_INDEX;
	wev->index = NGX_INVALID_INDEX;

	//data中保存连接
	rev->data = c;
	wev->data = c;
	wev->write = 1;
	return c;
}
```
然后来看epoll中的相关处理，epoll有两个相关的数据结构，epoll_event是每个epoll事件都会有一个这样的结构，它包含了epoll_data,它保存了相关的数据，其中我们关注epoll_data,它有一个域ptr是我们需要关注的，它保存了当前事件注册的连接。其实严格来说这个指针保存的是当前事件的连接指针的和事件instance域的一个组合值，这是因为对于大部分平台来说指针由于会对齐，因此最后几位都是0(也就是不使用的)，因此nginx就利用到连接指针的最后一位来保存事件的instance域.
```cpp
typedef union epoll_data {
	//保存连接指针的组合
	void *ptr;
	//句柄
	int fd;
	uint32_t u32;
	uint64_t u64;
} epoll_data_t;

struct epoll_event {
	uint32_t events;
	epoll_data_t data;
};
```
ok接下来就来看nginx是如何利用这个指针的,先来看ngx_epoll_add_event函数，这个函数就是加事件到epoll中。然后设置对应的epoll_data.
```cpp
static ngx_int_t
ngx_epoll_add_event(ngx_event_t *ev, ngx_int_t event, ngx_uint_t flags)
{
	int op;
	uint32_t events, prev;
	ngx_event_t *e;
	ngx_connection_t *c;
	struct epoll_event ee;

	ee.events = events | (uint32_t) flags;
	//最关键的一部分，可以看到设置connect 指针的最后一位为ev->instance.
	ee.data.ptr = (void *) ((uintptr_t) c | ev->instance);
	...
}
```
nginx还提供了一个将一个连接加入到epoll处理中的方法,因为很多时候是通过socket来get到连接，而此时我们只需要将get到的连接加入到epoll的处理队列就可以了。这里它的处理是和上面的处理差不多的。
```cpp
ngx_epoll_add_connection(ngx_connection_t *c)
{
	struct epoll_event ee;
	
	ee.events = EPOLLIN|EPOLLOUT|EPOLLET;
	//同样是将连接的read事件的instance标记加到指针的末尾。
	ee.data.ptr = (void *) ((uintptr_t) c | c->read->instance);

	ngx_log_debug2(NGX_LOG_DEBUG_EVENT, c->log, 0,           
				   "epoll add connection: fd:%d ev:%08XD", c->fd, ee.events);

	if (epoll_ctl(ep, EPOLL_CTL_ADD, c->fd, &ee) == -1) {
		ngx_log_error(NGX_LOG_ALERT, c->log, ngx_errno,            
					  "epoll_ctl(EPOLL_CTL_ADD, %d) failed", c->fd);
		return NGX_ERROR;
	}
	
	c->read->active = 1;
	c->write->active = 1;
	return NGX_OK;
}
```
然后是事件处理函数，当事件上报上来之后，我们来看nginx是如何处理以及利用instance这个标记的。 这里要注意的是当进入这个函数，instance会有两个值，一个是当add事件的时候保存在指针末尾的那个值，一个是当前connect对应的instance.
```cpp
static ngx_int_t
ngx_epoll_process_events(ngx_cycle_t *cycle, ngx_msec_t timer, ngx_uint_t flags)
{
	...  
	for (i = 0; i < events; i++) {
		//先从epoll添加事件时传递的数据中取出connect的地址。
		c = event_list[i].data.ptr;
		//然后取出来对应的instance值
		instance = (uintptr_t) c & 1;
		//然后取出来对应的connect指针(由于在添加是设置了最后一位指针，因此这里需要屏蔽掉最后一位).
		c = (ngx_connection_t *) ((uintptr_t) c & (uintptr_t) ~1);
	
		rev = c->read;
	
		//然后如果c-fd为－1则说明fd被关闭，如果rev->instance != instance则表示当前的fd已经被重新使用过了，也就是说这个event已经是stale的了，所以跳过这个事件，然后进行下一个
		if (c->fd == -1 || rev->instance != instance) {
			/*       
			* the stale event from a file descriptor         
			* that was just closed in this iteration     
			*/
			ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,              
						   "epoll: stale event %p", c);           
			continue;
		}
	...
}
```