# Nginx

## 1. 常用命令

```shell
nginx -s stop  快速关闭nginx，可能不保存相关信息，并迅速终止web服务
nginx -s quit  平稳关闭nginx，保存相关信息，有安排的结束web服务
nginx -s reload  因改变了Nginx相关配置，需要重新加载配置而重载
nginx -s reopen 重新打开日志文件
nginx -s filename 为 Nginx 指定一个配置文件，来代替缺省的
nginx -s test 不运行，仅仅测试配置文件。nginx 将检查配置文件的语法的正确性，并尝试打开配置文件中所引用到的文件
nginx -v 显示 nginx 的版本
nginx -V 显示 nginx 的版本，编译器版本和配置参数
```

## 2. 实战

### *a. 反向代理*

```shell
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
  worker_connections  1024;
}


http {
  include       mime.types;
  default_type  application/octet-stream;

  #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
  #                  '$status $body_bytes_sent "$http_referer" '
  #                  '"$http_user_agent" "$http_x_forwarded_for"';

  #access_log  logs/access.log  main;

  sendfile        on;
  #tcp_nopush     on;

  #keepalive_timeout  0;
  keepalive_timeout  65;

  #gzip  on;

  #上游服务器
  upstream my_server{
    server  localhost:8081;
    server  localhost:8082;
  }

  server {
    listen       8080;
    server_name  localhost;

    location / {
      root   html;
      index  index.html index.htm;
      proxy_pass http://my_server;
      proxy_set_header Host $host;
    }

  include servers/*;
}
```

### *b. 负载均衡*

在前面反向代理的基础上，可以拓展出请求上游服务器负载均衡的策略。

```shell
  #加权轮训 weight表示权值
  upstream my_server{
    server  localhost:8081  weight=5;  
    server  localhost:8082  weight=1;
  }
  
  ##最少连接
  upstream my_server{
    least_conn;
    
    server  localhost:8081;  
    server  localhost:8082;
  }
  
  #根据ip hash
  upstream my_server{
    ip_hash;
    
    server  localhost:8081;  
    server  localhost:8082;
  }
  
  #普通hash
  upstream my_server{
    hash $request_rui;
    
    server  localhost:8081;  
    server  localhost:8082;
  }
```

### *c. 静态站点*

todo...

```chshell

```





