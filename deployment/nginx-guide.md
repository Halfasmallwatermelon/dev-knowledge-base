# Nginx 配置指南

## 基础配置

```nginx
server {
    listen 80;
    server_name example.com;
    
    # 静态文件
    location / {
        root /var/www/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
    
    # API 代理
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

## HTTPS 配置

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    
    ssl_certificate /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    
    # 安全头
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    
    location / {
        root /var/www/html;
        try_files $uri $uri/ /index.html;
    }
}

# HTTP 重定向到 HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

## 负载均衡

```nginx
upstream backend {
    server backend1:3000;
    server backend2:3000;
    server backend3:3000;
    
    # 会话保持
    ip_hash;
}

server {
    location /api/ {
        proxy_pass http://backend;
    }
}
```

## 缓存

```nginx
# 静态资源缓存
location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
}

# 代理缓存
proxy_cache_path /tmp/nginx levels=1:2 keys_zone=my_cache:10m max_size=10g;

location /api/ {
    proxy_cache my_cache;
    proxy_cache_valid 200 10m;
    proxy_pass http://backend;
}
```

## 限流

```nginx
# 请求限制
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
    limit_req zone=api burst=20 nodelay;
    proxy_pass http://backend;
}
```

## Gzip 压缩

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml;
gzip_min_length 256;
```