![enter image description here](https://raw.githubusercontent.com/liexusong/nginx-0.1.0-comments/master/markdown/process.jpg)
一般来说nginx有以下几个core模块:
```cpp
* ngx_core_module
* ngx_errlog_module
* ngx_events_module
* ngx_http_module
```
ngx_events_module的ngx_events_block()会触发解析event{}配置块, 而ngx_http_module的ngx_http_block()会触发解析http{}配置块.  例如如下配置:
```nginx
user  nobody;
worker_processes  3;

#error_log  logs/error.log;
#pid        logs/nginx.pid;

events {
    connections  1024;
}

http {
    include       conf/mime.types;
    default_type  application/octet-stream;

    sendfile  on;

    #gzip  on;

    server {
        listen  80;

        charset         on;
        source_charset  koi8-r;

        #access_log  logs/access.log;

        location / {
            root   html;
            index  index.html index.htm;
        }
    }
}
```
