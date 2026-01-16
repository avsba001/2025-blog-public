# **前言**  
参考：[【虚拟组网】 ZeroTier/TailScale/Netbird/Easytier 点评](https://www.nodeseek.com/post-451418-1)

之前帖子里关于虚拟组网讨论很多，关于 Netbird 的自建教程网上比较笼统。这篇力求简单且全面，会比较长，估计要分 2–3 篇发出。需要一定动手能力与理解能力。本教程不讨论 DDNS、证书申请过程。

为了减少一键脚本报错导致调试困难，本教程环境为：国内环境 + 家宽 + 使用非 443 端口，采用 1panel 做图形化操作，简化部署。

---

## 前期准备

- 两个不同子域名（用于最大兼容性）：本例使用 `sso.xxx.com` 和 `netbird.xxx.com`，并通过 DDNS 指向你的公网 IP。
- 安装 1panel（示例一键安装脚本）：
  ```bash
  bash -c "$(curl -sSL https://resource.fit2cloud.com/1panel/package/v2/quick_start.sh)"
  ```
- 系统：Debian 12，至少 1GB 内存。
- 已准备好证书（必须项）：可以通过 1panel 上传/申请证书。

---

## 一、容器安装（在 1panel 中进行）

1. 进入 1panel 面板 → 应用商店 → 更新远程应用。  
   ![更新远程应用](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1e1a28-2x.png)

2. 安装 OpenResty  
   - 安装时把 HTTPS 端口改为你要的端口，本教程使用 `1443`。  
   ![OpenResty 配置端口](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1e4whk-iv.png)

3. 安装 PostgreSQL 与 ZITADEL  
   - PostgreSQL 直接安装，用户名/密码可以不用记（面板管理即可）。

4. 安装 ZITADEL 时需要修改以下配置项：  
   - 管理秘钥（必须 32 字节！）  
   - 外部域名：本教程中为 `sso.xxx.com`  
   - 组织管理员：`admin@sso.xxx.com`，并指定组织管理员密码  
   - 添加镜像加速：勾选并把第三行改为：
     ```
     image: ghcr.nju.edu.cn/zitadel/zitadel:v3.3.2
     ```
   - 勾选“编辑 compose 文件”，确认后安装。  
   ![Zitadel 安装配置](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1eh7g3-zs.png)  
   ![Zitadel 安装确认](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1enxv3-qu.png)

---

## 二、添加反向代理并配置 HTTPS（OpenResty/Nginx）

> 注意：红框处记得修改为你的域名/端口。如果是手动上传的证书，面板里自动选择项可能不可用。

1. 在 1panel 的反向代理配置中启用 HTTPS，选择你上传/申请的证书。  
   ![反向代理配置示意](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1esh7i-wz.png)

2. 启用 HTTPS 后选择证书：  
   ![选择证书](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1evw0w-zc.png)  
   ![证书选择确认](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1ex0sn-g6.png)

3. 在反向代理 → root → 源文（即 location /）中，修改为以下参数（示例 Nginx 配置）：
```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header Host $host:$server_port;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header REMOTE-HOST $remote_addr;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Port $server_port;
    proxy_http_version 1.1;
    add_header X-Cache $upstream_cache_status;
    add_header Cache-Control no-cache;
    proxy_ssl_server_name off;
    proxy_ssl_name $proxy_host;
}
```
保存配置。

4. 非常重要：点击“配置文件”，**删除**以下这段（会干扰 OIDC 的 well-known）：
```nginx
location ^~ /.well-known {
    allow all;
    root /usr/share/nginx/html;
}
if ( $uri ~ "^/\.well-known/.*\.(php|jsp|py|js|css|lua|ts|go|zip|tar\.gz|rar|7z|sql|bak)$" ) {
    return 403;
}
```

---

## 三、路由器端口转发

- 把外网对应端口（本教程为 `1443`）转发到部署主机。路由器转发方法因设备不同，请自行查阅设备说明或搜索教程。

---

## 四、登录 Zitadel（sso.xxx.com）

- 使用你在安装时配置的组织管理员账号（例如：`admin@sso.xxx.com`）和密码登录，登录后请修改密码。  
  ![Zitadel 登录示意](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/09/16/1fe4ff-ua.png)

---

## 五、配置 Netbird 的 IDP（在 Zitadel 中）

下面的步骤基本是官方文档流程（已中文化、简化）。最终会得到三个“考点”用于后续 Netbird 配置（Client ID、Client Secret、OIDC 配置端点）。

### 1. 在 Zitadel 中创建 Project
- 控制台 → Projects → Create New Project
- 填写：
  - Name：`NETBIRD`
- 点击 Continue。  
  ![创建 Project](https://docs.netbird.io/docs-static/img/selfhosted/identity-providers/self-hosted/zitadel/zitadel-new-application.png)

### 2. 在 Project 下创建 Application
- 选择项目 `NETBIRD`
- Applications → New
- 填写：
  - Name：`netbird`
  - Application type：`User Agent`
- 认证方法选择 `PKCE`
- 点击 Continue

在表单中填写（Redirect / Logout URIs）：
- Redirect URIs:
  - `https://netbird.xxx.com:1443/auth`
  - `https://netbird.xxx.com:1443/silent-auth`
  - `http://localhost:53000`
- Post Logout URIs:
  - `https://netbird.xxx.com:1443/`
  - `https://sso.xxx.com:1443/`

完成并创建后，会出现第一个考点：ClientId（记住它，后面 Netbird 部署需要用到）

创建后进入 Configuration，修改 Grant Types：
- 勾选：Authorization Code、Device Code、Refresh Token，保存。

进入 token settings：
- Auth Token Type：JWT
- 勾选：Add user roles to the access token
- 保存。  
  ![token settings 示例](https://docs.netbird.io/docs-static/img/selfhosted/identity-providers/self-hosted/zitadel/zitadel-token-settings.png)

### 3. 创建 Service User（Netbird 服务用户）
- 顶部菜单 → Users → Service Users → New
- 填写：
  - User Name：`netbird`
  - Name：`netbird`
  - Description：`Netbird Service User`
  - Access Token Type：`JWT`
- 点击 Create  
  ![create service user](https://docs.netbird.io/docs-static/img/selfhosted/identity-providers/self-hosted/zitadel/zitadel-create-user.png)

创建后：
- Action → Generate Client Secret
- 复制弹窗中的 ClientSecret（这是第二个考点，重要：复制并妥善保管，别重复生成）  
  ![service user secret](https://docs.netbird.io/docs-static/img/selfhosted/identity-providers/self-hosted/zitadel/zitadel-service-user-secret.png)

### 4. 授予 netbird 服务用户“组织用户管理员”角色
- 菜单 → Organization
- 右上角 `+` → 搜索 `netbird` → 勾选 `Org User Manager` → Add

### 5. 获取 OIDC 配置端点（第三个考点）
- OIDC 配置地址（用于 NETBIRD_AUTH_OIDC_CONFIGURATION_ENDPOINT）：
  ```
  https://sso.xxx.com:1443/.well-known/openid-configuration
  ```

---

## 三个“考点”总结（部署 Netbird 时需要用到）

在下一篇的 `setup.env` 中需要填入这些值：

- 第三个考点（OIDC 配置端点）：
  ```env
  NETBIRD_AUTH_OIDC_CONFIGURATION_ENDPOINT="https://sso.xxx.com:1443/.well-known/openid-configuration"
  ```

- 第一个考点（Client ID，从创建的 Application 中获取）：
  ```env
  NETBIRD_AUTH_CLIENT_ID="（Application 的 ClientId）"
  NETBIRD_AUTH_AUDIENCE="（同上，通常使用 ClientId）"
  NETBIRD_AUTH_DEVICE_AUTH_CLIENT_ID="（同上）"
  NETBIRD_AUTH_DEVICE_AUTH_AUDIENCE="（同上）"
  ```

- 第二个考点（Service User 的 Client Secret）：
  ```env
  NETBIRD_IDP_MGMT_CLIENT_SECRET="（Service User 的 ClientSecret）"
  ```

示例 env 片段（请在实际文件中替换考点内容）：
```env
NETBIRD_AUTH_OIDC_CONFIGURATION_ENDPOINT="第三个考点"
NETBIRD_USE_AUTH0=false
NETBIRD_AUTH_CLIENT_ID="第一个考点"
NETBIRD_AUTH_SUPPORTED_SCOPES="openid profile email offline_access api"
NETBIRD_AUTH_AUDIENCE="第一个考点"
NETBIRD_AUTH_REDIRECT_URI="/auth"
NETBIRD_AUTH_SILENT_REDIRECT_URI="/silent-auth"

NETBIRD_AUTH_DEVICE_AUTH_PROVIDER="hosted"
NETBIRD_AUTH_DEVICE_AUTH_CLIENT_ID="第一个考点"
NETBIRD_AUTH_DEVICE_AUTH_AUDIENCE="第一个考点"

NETBIRD_MGMT_IDP="zitadel"
NETBIRD_IDP_MGMT_CLIENT_ID="netbird"
NETBIRD_IDP_MGMT_CLIENT_SECRET="第二个考点"
NETBIRD_IDP_MGMT_EXTRA_MANAGEMENT_ENDPOINT="https://sso.xxx.com:1443/management/v1"
NETBIRD_MGMT_IDP_SIGNKEY_REFRESH=true
```

以上参数将在下一篇（Netbird 容器部署与 setup.env）中详细说明。

---

## 补充问答

**Q：为什么要两个域名？别人明明一个域名也能部署。**  
A：Zitadel 是一个通用的认证工具，可能被多个服务复用（例如 Vaultwarden 等）。把认证（sso.xxx.com）和 Netbird（netbird.xxx.com）分开，能最大化兼容性与隔离性。~~我不会说因为我嫌反代麻烦,太菜不会玩~~

鸡腿催更我第二篇，嘿嘿
---

## 参考资料

- 1Panel 部署 NetBird：实现内网穿透+零信任组网（知乎专栏）  
  https://zhuanlan.zhihu.com/p/1931065645635724706
- Netbird 官方文档  
  https://netbird.io/
```