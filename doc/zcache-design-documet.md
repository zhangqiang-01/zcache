# zcache设计

## 申明

zcache设计理念及相关代码为本作者独自构思和编写，属于兴趣类项目，不涉及公司相关研发机密及专利等。如果您有任何疑问，可以与我联系。


Author: zhangqiang.sunny

Company: bytedance

Phone number: 13260255349


## 简介

zcache是一个基于nginx实现的缓存服务。它采用单盘单进程模式，每个zcache都会监听一个唯一的端口，用于提供缓存服务。

zcache同样是一个proxy cache, 当然也会提供相关的http restful api, 使之成为一个纯cache服务。

zcache包含对nginx的部分侵入性改造(主要是nginx进程管理，改动的代码量不超过100行)，一个cache module (它是一个core module, 等同于http module)，以及一个http content module。其中cache module用于实现缓存功能初始化和缓存读写等工作；content module是以content handler的方式介入nginx原生的请求处理流程，相当于重写了nginx proxy，用于实现缓存读写以及合并回源。

zcache优点:
- 经典proxy cache模型，也可扩展为存储模型。
- 对nginx代码侵入较少，不改动原生nginx处理流程，仅做少量的适配工作。
- 复用nginx绝大部分功能，如配置管理、指令及功能等。
- 业务开发简单，利用nginx插件、nginx lua进行业务开发。
- 后续可考虑扩展nginx lua, 让nginx lua参与到zcache_content_handler的处理当中。


## zcache设计

### 配置设计
```
cache {
    listen 8081; # 与nginx server listen指令使用一致
    store path=/dev/sda mem_size=2048M;
    ... other cache conf ...
}

cache {
    listen 8082; # 与nginx server listen指令使用一致
    store path=/dev/sdb mem_size=2048M;
    ... other cache conf ...
}

http {
    server {
        listen 8999;  # 无任何意义, 仅仅为了满足nginx配置需求
        server_name www.a.com;
        ....
        content_by_cache on;
        cache_proxy_pass http://upstream;
    }

    server {
        listen 8999; # 无任何意义, 仅仅为了满足nginx配置需求
        server_name www.b.com;
        ....
        content_by_cache on;
        cache_proxy_pass http:
    }
}
```

基于上述配置, nginx会创建两个zcache worker, 这些zcache worker独立于原生的nginx worker，它们的个数取决于cache conf block的个数。并且这两个worker分别listen了8081、8082端口，向外提供相关的http处理能力。


域名配置管理仍然是nginx http server conf, 所有的使用、功能等都沿用nginx http server conf, 这里的8999端口实际上是一个傀儡，在zcache worker该端口已经被关闭。zcache从对应的cache port accept到新的连接后会与nginx原生的http accepter对接，全部继承nginx的协议处理能力，管理员可以选择使用http1.1、http2.0或quic等协议接入zcache。


### 示意图

缓存集群:

![](https://github.com/zhangqiang-01/zcache/raw/master/doc/img/1.png)

进程示意图:

![](https://github.com/zhangqiang-01/zcache/raw/master/doc/img/2.png)

请求处理:

![](https://github.com/zhangqiang-01/zcache/raw/master/doc/img/3.png)


在上述的配置中，我们为了兼容nginx原有的配置解析流程，仍然使用8999端口进行配置管理，但是并不期望8999端口向外提供服务，因此在进程初始化阶段该端口就会被close。同时在8081端口的listening对象创建时，会将它的server对象绑定到8999，从而复用8999端口下的域名配置。


### zcache进程管理

zcache会创建属于自己的zcache worker process, 它遵守原生的nginx restart/ reload/ reopen等操作，并且实现与nginx worker基本一致。只是zcache worker是有状态的，并且不允许多个worker同时操作磁盘，因此在zcache worker初始化时，我们会先从共享内存中获取对应cache item的锁，如果上锁失败，则初始化流程将被block, 直到上锁成功或超时退出。

zcache worker有自己的process_number, 这区别与nginx worker_process。同时zcache有自己的quit_handler函数，用于实现缓存的优雅退出。


### zcache模块管理
zcache目前有三个模块，分别是ngx_zcache_module, ngx_zcache_core_module, ngx_zcache_http_content_module。

- ngx_zcache_module: NGX_CORE_MODULE
- ngx_zcache_core_module: NGX_ZCACHE_MODULE
- ngx_zcache_http_content_module: NGX_HTTP_MODULE
