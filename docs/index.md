  ngx_init_cycle()函数解释:
  =========================
  * ngx_init_cycle()函数的作用是初始化一个新的生命周期,
  * 在ngx_init_cycle()函数中会遍历所有的core模块,
  * 并调用其create_conf()接口来创建配置上下文.
  * 属于core模块的有5个, 如下:
  *   ngx_core_module, ngx_errlog_module, ngx_conf_module, ngx_events_module, ngx_http_module
  * 接着ngx_init_cycle()函数会调用ngx_conf_parse()函数来解析配置文件(下面会细说).
  * 解析完配置文件后, 遍历所有的core模块, 并调用其init_conf()接口来初始化配置上下文.
  * 然后nginx会遍历所有模块(注意是所有载入到nginx的模块), 并调用其init_module()接口来初始化模块.
  * 当然除此之外, ngx_init_cycle()函数还会根据配置文件的配置来打开要监听的IP和端口.
  
  ngx_conf_parse()函数解释:
  ========================
  * 