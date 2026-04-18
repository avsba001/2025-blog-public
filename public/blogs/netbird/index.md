之前写了一篇netbird的IDP部署教程，好巧不巧，Netbird在0.65.0版本内置了IDP，现在部署教程可以更简单了.

# 一、基础需求
1. 1panel及自带的OpenResty
2. 自备域名以及SSL证书,并且1panel面板开启ssl
3. ~一个逻辑清晰的🧠~

# 二、创建所需目录并修改配置文件(合并部署)

## 1. 使用1panel在`/opt/1panel/docker/compose/`创建目录netbird
*或者*
使用命令
```mkdir /opt/1panel/docker/compose/netbird```

## 2.在netbird目录下创建`config.yaml`文件
写入
```yaml
# NetBird 一体化服务器配置
# 将此文件复制为 config.yaml 并根据你的部署环境修改
#
# 这是一个 Management 服务器，同时可内置 Signal、Relay、STUN 服务
# 默认所有服务本地运行，也可以通过配置改为使用外部服务
#
# 架构说明：
#   - Management：始终本地运行（当前就是管理服务器）
#   - Signal：默认本地运行；设置 'signalUri' 后改为外部（本地关闭）
#   - Relay：默认本地运行；设置 'relays' 后改为外部（本地关闭）
#   - STUN：默认 3478 端口本地运行；设置 'stuns' 后改为外部

server:
  # 主监听端口（HTTP/gRPC，包含 Management、Signal、Relay）代表容器内端口，可以不修改
  listenAddress: ":80"

  # 对外暴露地址（客户端连接使用）代表对外端口，必须修改，比如4443
  # 用于 relay 连接和管理 DNS
  # 格式：协议://域名:端口（例如：https://server.mycompany.com:4443）
  exposedAddress: "https://server.mycompany.com:4443"

  # STUN 端口（默认 3478；如果使用外部 STUN 则无需配置）
  stunPorts:
    - 3478

  # 指标监控端口（Prometheus 等）
  metricsPort: 9090

  # 健康检查接口
  healthcheckAddress: ":9000"

  # 日志配置
  logLevel: "info"     # 日志级别：panic, fatal, error, warn, info, debug, trace
  logFile: "console"   # 输出到控制台或文件路径

  # Relay 认证密钥（使用内置 relay 必填）
  authSecret: "9k5P1EpfMMeFXzvsJjrtzd8nakY6QqQ4hkw5LSAN3NU=" #加密密钥openssl rand -base64 32 生成

  # 数据目录（数据库、状态等）
  dataDir: "/var/lib/netbird/"

  # =====================================================================
  # 外部服务覆盖（可选）
  # 设置后将关闭对应本地服务
  # =====================================================================

  # 外部 STUN（配置后禁用本地 STUN）
  # stuns:
  #   - uri: "stun:stun.example.com:3478"

  # 外部 Relay（配置后禁用本地 Relay）
  # relays:
  #   addresses:
  #     - "rels://relay.example.com:443"
  #   credentialsTTL: "12h"
  #   secret: "9k5P1EpfMMeFXzvsJjrtzd8nakY6QqQ4hkw5LSAN3NU=" #加密密钥openssl rand -base64 32 生成

  # 外部 Signal（配置后禁用本地 Signal）
  # signalUri: "https://signal.example.com:443"

  # =====================================================================
  # 管理服务配置
  # =====================================================================

  # 是否关闭匿名统计
  disableAnonymousMetrics: true
  disableGeoliteUpdate: true

  # 内置认证系统（Dex / OIDC）
  auth:
    # OIDC 发行者地址（必须公网可访问）
    issuer: "https://server.mycompany.com:4443/oauth2"
    localAuthDisabled: false
    signKeyRefreshEnabled: false

    # Dashboard 登录回调地址
    dashboardRedirectURIs:
      - "https://server.mycompany.com:4443/nb-auth"
      - "https://server.mycompany.com:4443/nb-silent-auth"

    # CLI 登录回调地址
    cliRedirectURIs:
      - "http://localhost:53000/"

    # 初始管理员（可选）
    # owner:
    #   email: "admin@example.com"
    #   password: "initial-password"

  # 数据存储配置
  store:
    engine: "sqlite"   # sqlite / postgres / mysql
    encryptionKey: "zT0uM/9tKXBuJCXS+mwihttNIaB2obn2FeZ14UqEdcY="  # 加密密钥openssl rand -base64 32 生成

  # 活动日志存储（可选）
  # activityStore:
  #   engine: "sqlite"
  #   dsn: ""

  # 认证数据库（可选）
  # authStore:
  #   engine: "sqlite3"
  #   dsn: ""

  reverseProxy:
    trustedHTTPProxies:
      - "0.0.0.0/0"
```
## 3.在netbird目录下创建`dashboard.env`
写入
```yaml
# Endpoints
NETBIRD_MGMT_API_ENDPOINT=https://server.mycompany.com:4443
NETBIRD_MGMT_GRPC_API_ENDPOINT=https://server.mycompany.com:4443
# OIDC - using embedded IdP
AUTH_AUDIENCE=netbird-dashboard
AUTH_CLIENT_ID=netbird-dashboard
AUTH_CLIENT_SECRET=
AUTH_AUTHORITY=https://server.mycompany.com:4443/oauth2
USE_AUTH0=false
AUTH_SUPPORTED_SCOPES=openid profile email groups
AUTH_REDIRECT_URI=/nb-auth
AUTH_SILENT_REDIRECT_URI=/nb-silent-auth
# SSL
NGINX_SSL_PORT=4443
# Letsencrypt
LETSENCRYPT_DOMAIN=none

```

# 三、在1panel中创建编排

![image](https://cdn.nodeimage.com/i/Gs3KZlCx9vgvbUq3blWyTkeqLCxhxmjz.png)

写入
```yaml
services:
  # UI dashboard，已替换源为汉化版，官方原版为netbirdio/dashboard:latest
  dashboard:
    image: docker.1ms.run/netbird-dashboard:dev
    container_name: netbird-dashboard
    restart: unless-stopped
    networks: [netbird]
    ports:
      - '127.0.0.1:8080:80'
    env_file:
      - ./dashboard.env
    logging:
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "2"

  # Combined server (Management + Signal + Relay + STUN)
  netbird-server:
    image: docker.1ms.run/netbirdio/netbird-server:latest
    container_name: netbird-server
    restart: unless-stopped
    networks: [netbird]
    ports:
      - '127.0.0.1:8081:80'
      - '3478:3478/udp'
    volumes:
      - netbird_data:/var/lib/netbird
      - ./config.yaml:/etc/netbird/config.yaml
    environment:
      - NB_DISABLE_GEOLOCATION=true
    command: ["--config", "/etc/netbird/config.yaml"]
    logging:
      driver: "json-file"
      options:
        max-size: "20m"
        max-file: "2"

volumes:
  netbird_data:

networks:
  netbird:
```

# 四、在openresty中创建反向代理

![image](https://cdn.nodeimage.com/i/UvpaQsWkpqD4bH8DWzxx7PlekfOWQKyV.png)

## 1.因openresty无法ui设置grpc，需要直接写入配置文件，以下为内容

![image](https://cdn.nodeimage.com/i/VFLIQy22TGN10Zo0IFuBjmXiYuBbwdW1.png)

```
    proxy_set_header X-Real-IP $remote_addr; 
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
    proxy_set_header X-Scheme $scheme; 
    proxy_set_header X-Forwarded-Proto https; 
    proxy_set_header X-Forwarded-Host $host:$server_port; 
    proxy_set_header X-Forwarded-Port $server_port; 
    grpc_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
    location ^~ /management.ManagementService/ {
        grpc_read_timeout 300s; 
        grpc_send_timeout 300s; 
        grpc_socket_keepalive on; 
        grpc_pass grpc://127.0.0.1:8081; 
        access_log off; 
    }
    location ^~ /signalexchange.SignalExchange/ {
        grpc_read_timeout 300s; 
        grpc_send_timeout 300s; 
        grpc_socket_keepalive on; 
        grpc_pass grpc://127.0.0.1:8081; 
        access_log off; 
    }
```

## 2.写入更多反代（如图）

![image](https://cdn.nodeimage.com/i/wTuHrnSRbdGBYhcodNbzpSFWkywDnsBT.png)
示例
```
location ^~ /oauth2 {
    proxy_pass http://127.0.0.1:8081; 
    proxy_set_header Host $host:$server_port; 
    proxy_set_header X-Real-IP $remote_addr; 
    proxy_set_header X-Forwarded-Host $host:$server_port; 
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
    proxy_set_header REMOTE-HOST $remote_addr; 
    proxy_set_header Upgrade $http_upgrade; 
    proxy_set_header Connection $http_connection; 
    proxy_set_header X-Forwarded-Proto $scheme; 
    proxy_set_header X-Forwarded-Port $server_port; 
    proxy_http_version 1.1; 
    add_header X-Cache $upstream_cache_status; 
    proxy_ssl_server_name off; 
    proxy_ssl_name $proxy_host; 
}
```

# 五、大功告成，附上 面板汉化教程
将第三步中的`netbirdio/dashboard:latest`改为`docker.1ms.run/netbird-dashboard:dev`
附上大佬的仓库
[Kunlun-Hub/dashboard-2.36.0](https://github.com/Kunlun-Hub/dashboard-2.36.0)

![image](https://cdn.nodeimage.com/i/js0zIqtYZn5pHfTuZ81CirYd3UXspfzN.png)


# 六、分离部署stun和relay（可选）
同样使用1panel部署
## 1.创建docker compose
```
services:
  relay:
    image: docker.1ms.run/netbirdio/relay:latest
    container_name: netbird-relay
    restart: unless-stopped
    ports:
      - 11276:443
      # Expose all ports listed in NB_STUN_PORTS
      - 11277:11277/udp
    env_file:
      - relay.env
    volumes:
      - relay_data:/data
      - /opt/1panel/secret:/etc/cert
    logging:
      driver: json-file
      options:
        max-size: "120m"
        max-file: "2"

volumes:
  relay_data:
```
## 2.创建`relay.env`文件
```
NB_LOG_LEVEL=info
NB_LISTEN_ADDRESS=:443
NB_EXPOSED_ADDRESS=rels://xxxxx.com:11276
NB_AUTH_SECRET="与管理端相同"

# TLS via Let's Encrypt (automatic certificate provisioning)
#NB_LETSENCRYPT_DOMAINS=xxxxx.com
NB_TLS_CERT_FILE=/etc/cert/server.crt
NB_TLS_KEY_FILE=/etc/cert/server.key
# Embedded STUN (comma-separated for multiple ports, e.g., 3478,3479)
NB_ENABLE_STUN=true
NB_STUN_PORTS=11277
```
## 3.同时修改管理端的`config.yaml`
```
  # 外部 STUN（配置后禁用本地 STUN）
  stuns:
    - uri: "stun:xxxx.com:3478"

  # 外部 Relay（配置后禁用本地 Relay）
  relays:
    addresses:
      - "rels://xxxx.com:443"
    credentialsTTL: "12h"
    secret: "9k5P1EpfMMeFXzvsJjrtzd8nakY6QqQ4hkw5LSAN3NU=" #加密密钥openssl rand -base64 32 生成
```

# 七、常见问题答疑
问：1.stun端口怎么修改？
答：把所有3478的字样改成你需要的就可以，比如10086。

问：2.为什么你的中继 docker compose多映射了`- /opt/1panel/secret:/etc/cert` 目录
答：个人小巧思，利用1panel面板可用证书作为Netbird加密证书（必须是可用）

问：3.如何部署多个中继(relay)
答：重复中继部署教程，然后服务端`config.yaml`这样修改下
```
  relays:
    addresses:
      - "rels://hostname:port"  # TLS
      - "rel://hostname:port"   # Plain (not recommended)
    secret: "shared-secret"
    credentialsTTL: "24h"  # How long relay credentials are valid
```