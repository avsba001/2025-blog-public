## 前言

国内 VPS 中大量是 **禁止 UDP 的 NAT 机器**。  
这些机器在常规使用下峰值带宽高、性价比也不错，但用于内网穿透做中转时却因为 **无 UDP** 而 **无法连接**（比如第三方的 Tailscale + 自建 DERP）。

那么，我们如何利用 **EasyTier 的特性**，让这类 VPS 也能用来加速、绕过限制、以及应对部分敏感场景呢？

本教程适用面非常广，可解决：

1. **部分企业内网管控导致 EasyTier API 域名被禁的问题**  
   （比如我自己的 `https://config-server.easytier.cn` 被移动直接干掉）
2. **避免非中心化部署时大量机器修改配置的麻烦**
3. **自建 Web 控制台，一套配置占用 1 个端口完成 HTTP + WSS 转发**
4. **家宽部署、非 443 端口承载**

---

## 📌 使用前准备

1. 一份 **SSL 证书**
2. 一台 **无 UDP 的 VPS**
3. 一台 **能正常访问公网的 Linux 主机**（家宽也行）
4. 一个 **可用域名**

---

# 第一部分：WEB 控制台搭建

## 一、安装 Web 控制台

### 1. Web 主机安装 1Panel（可选）
1Panel 能提供可视化 Nginx、证书管理，适合新手。

📌 地址：  
https://1panel.cn/

---

### 2. 下载 EasyTier（Linux）
可手动下载解压至 `/root/`  
或直接使用命令：

```
wget https://ghfast.top/https://github.com/EasyTier/EasyTier/releases/download/v2.4.5/easytier-linux-x86_64-v2.4.5.zip
```

---

### 3. 解压

```
unzip easytier-linux-x86_64-v2.4.5.zip
```

解压后结构如下：

```
/root/easytier-linux-x86_64/easytier-web-embed
```

---

### 4. 注册为 systemd 服务（自启）

编辑 service 文件：

```
vim /etc/systemd/system/easytier-web-embed.service
```

写入：

```
[Unit]
Description=Easytier Web Embed

[Service]
ExecStart=/root/easytier-linux-x86_64/easytier-web-embed --config-server-port 11211 --api-server-port 22020 -p ws
Restart=always

[Install]
WantedBy=multi-user.target
```

启用：

```
systemctl daemon-reload
systemctl enable easytier-web-embed
systemctl start easytier-web-embed
```

### 参数说明

```
--config-server-port   # ET 客户端与 Web 控制台的通讯端口
--api-server-port      # 网页端访问的端口
-p ws                  # 强制使用 WebSocket（TCP），默认是 UDP
```

打开浏览器访问：

```
http://服务器IP:22020
```

如果能打开 Web，就是成功了。

---

## 二、1Panel 新建网站 + 反向代理

证书申请自行完成。

### 反代示意图👇

![wechat_2025-11-21_190851_279.png](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/11/21/15im5j-yf.png)
![wechat_2025-11-21_190953_596.png](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/11/21/15im3b-za.png)
![wechat_2025-11-21_191151_636.png](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/11/21/15inlw-z9.png)

---

### **⚠️ 重头戏：修改 Nginx 配置**

进入网站 → **配置文件**  
**删除这一行：**

```
include /www/sites/xxxx.com/proxy/*.conf;
```

然后手动粘贴下面内容（不要全删！！只删 include 那行）

```
#加入到server块前面，注意缩进
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

upstream easytier_http_backend {
    server 127.0.0.1:22020;
}

upstream easytier_ws_backend {
    server 127.0.0.1:11211;
}

server {

#上面不变，删除 `include /www/sites/xxxx.com/proxy/*.conf;` 这一行！！

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
```

---

至此，你已经可以访问：

```
https://你的域名:1443
```

并在 网页的 api host填入：

```
https://你的域名:1443
```

---

# 第二部分：客户端接入自建 Web

## 1. 客户端注册机器

按照官方教程操作，只需要把注册命令修改为：

```
easytier-cli service install -- -w wss://<域名>:1443/<注册时的用户名>
```

这样客户端会通过 WSS → Nginx → EasyTier Web 控制台  
全程无需 UDP  
完美适配无 UDP 的 VPS。

---

## 2. 在 Web 控制台检查上线情况

登录自己的控制台  
查看节点是否已成功在线。

---