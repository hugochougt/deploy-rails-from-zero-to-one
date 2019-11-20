# Config NGINX

配置好部署脚本之后，就可以开始登录服务器，配置 NGINX 了。将域名信息改为你的域名，对应的文件目录也要一并更改。

## 配置 NGINX conf

```
upstream rails-deployment-demo-puma {
    server unix:///var/www/rails-deployment-demo/shared/tmp/sockets/puma.sock fail_timeout=0;
}
server {
    listen 443 ssl;
    server_name deploy-demo.zq-dev.com;

    ssl_certificate /etc/letsencrypt/live/deploy-demo.zq-dev.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/deploy-demo.zq-dev.com/privkey.pem;
    ssl_session_cache shared:le_nginx_SSL:1m;
    ssl_session_timeout 1440m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers EECDH+ECDSA+AESGCM:EECDH+aRSA+AESGCM:EECDH+ECDSA+SHA512:EECDH+ECDSA+SHA384:EECDH+ECDSA+SHA256:ECDH+AESGCM:ECDH+AES256:DH+AESGCM:DH+AES256:RSA+AESGCM:!aNULL:!eNULL:!LOW:!RC4:!3DES:!MD5:!EXP:!PSK:!SRP:!DSS;

    root /var/www/rails-deployment-demo/current/public;
    try_files $uri/index.html $uri.html $uri @app;
    client_max_body_size 4G;
    keepalive_timeout 120;

    access_log /var/www/rails-deployment-demo/shared/log/nginx.access.log combined;
    error_log /var/www/rails-deployment-demo/shared/log/nginx.error.log  error;
    error_page 500 502 503 504 /50x.html;

    gzip on;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/vnd.ms-fontobject application/x-font-ttf font/opentype image/svg+xml image/x-icon;
    gzip_min_length 1000;

    location @app {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto $scheme;
        proxy_set_header  X-Forwarded-Ssl on;
        proxy_set_header  X-Forwarded-Port $server_port;
        proxy_set_header  X-Forwarded-Host $host;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_pass http://rails-deployment-demo-puma;
    }
    location ^~ /assets/ {
        expires max;
        gzip_static on;
        add_header Pragma public;
        add_header Cache-Control public;
    }
}

server {
    listen 80;
    server_name deploy-demo.zq-dev.com;
    location /.well-known/acme-challenge {
      root /tmp;
    }
    location / {
      return 302 https://$server_name$request_uri;
    }
}
```

## 配置 Let's Encrypt 证书

因为 HTTP 协议都是明文传输，一个正常的网站，肯定要使用 HTTPS 协议访问的。我们使用免费的 [Let's Encrypt](https://letsencrypt.org/) 证书加密请求流量。
