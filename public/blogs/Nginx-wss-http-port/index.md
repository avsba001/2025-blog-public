```
# ------------------------
# HTTP -> HTTPS 跳转
# ------------------------
server {
    listen 80;
    listen [::]:80;
    server_name xxxx;
    return 301 https://$host$request_uri;
}

# ------------------------
# HTTPS + HTTP3 + HTTP/WS 分流
# ------------------------
# 1. map Upgrade 头
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

# 2. upstream
# 2. upstream 定义
upstream easytier_http_backend {
    server 192.168.2.210:22020;
}

upstream easytier_ws_backend {
    server 192.168.2.210:11211;
}

server {
    listen 5443 ssl;
    listen [::]:5443 ssl;
    server_name xxxx;

    http2 on;   # 开启 HTTP/2
    server_name xxxx;

    root /www/sites/xxxx/index;
    index index.php index.html index.htm default.php default.htm default.html;

    access_log /www/sites/xxxxx/log/access.log main;
    error_log /www/sites/xxxx/log/error.log;

    # TLS 配置
    ssl_certificate /www/sites/xxxx/ssl/fullchain.pem;
    ssl_certificate_key /www/sites/xxxx/ssl/privkey.pem;
    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:!aNULL:!eNULL:!EXPORT:!DSS:!DES:!RC4:!3DES:!MD5:!PSK:!KRB5:!SRP:!CAMELLIA:!SEED;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    add_header Strict-Transport-Security "max-age=31536000";
    add_header Alt-Svc 'h3=":443"; ma=2592000';

    # ------------------------
    # 隐私/安全路径屏蔽
    # ------------------------
    location ~ ^/(\.user.ini|\.htaccess|\.git|\.env|\.svn|\.project|LICENSE|README.md) {
        return 404;
    }
    location ^~ /.well-known {
        allow all;
        root /usr/share/nginx/html;
    }
    if ($uri ~ "^/\.well-known/.*\.(php|jsp|py|js|css|lua|ts|go|zip|tar\.gz|rar|7z|sql|bak)$") {
        return 403;
    }


    location / {
        # WebSocket 请求分流
        set $backend easytier_http_backend;
        if ($http_upgrade = "websocket") {
            set $backend easytier_ws_backend;
        }

        proxy_pass http://$backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Sec-WebSocket-Protocol $http_sec_websocket_protocol;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
        proxy_buffering off;
    }
}
