```bash

user nobody;
worker_processes auto;

#error_log  /opt/var/log/nginx/error.log;
#error_log  /opt/var/log/nginx/error.log  notice;
#error_log  /opt/var/log/nginx/error.log  info;

#pid        /opt/var/run/nginx.pid;


events {
  worker_connections 10240;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  /opt/var/log/nginx/access.log main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    # server {
    #     listen       80;
    #     server_name  www.leson.pro;

    #     location / {
    #         root   /opt/share/nginx/html;
    #         index  index.html index.htm;
    #     }
    #     error_page   500 502 503 504  /50x.html;
    #     location = /50x.html {
    #         root   html;
    #     }

    # }

    # server {
    #   listen 80;
    #   server_name www.leson.pro;
    #   rewrite ^(.*)$ https://$host$1 permanent;
    # }
    server {
      listen  443 ssl;
      server_name www.leson.pro;
     
      ssl_certificate /opt/usr/ssl/6181812_www.leson.pro.pem;
      ssl_certificate_key /opt/usr/ssl/6181812_www.leson.pro.key; 
      ssl_ciphers EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH+ECDSA+3DES:EECDH+aRSA+3DES:RSA+3DES:!MD5;
      ssl_prefer_server_ciphers   on;
     
      location / {
          proxy_set_header Host          $host;
          proxy_set_header X-Real-IP     $remote_addr;
          proxy_set_header X-Forward-For $remote_addr;
          proxy_set_header X-Forward-For $proxy_add_x_forwarded_for;
          proxy_pass http://192.168.2.1/;
      }
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

```

```bash
# 修改文件句柄限制
ulimit -SHn 20480
```

