### 前言：
 Remnawave 自带了订阅网页的docker，但是网页经常忘了存，得从代理软件里面复制非常不方便对不对。现在有很方便的telegram 机器人，一键跳转到订阅页面   ~~（真的方便嘛，反正帅就完事了）~~

本教程不含如何申请域名，如何申请https证书，如何反代。

# 1.获取Telegram 用户ID

和这个 [getmyid_bot](https://t.me/getmyid_bot) 聊天 输入`/start `
`Your ID:` 就是你的ID。

# 2.在面板中填入Telegram 用户ID
![QQ_1766856020080.png](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/91529a9d2fa65a01ac8d1a24f2d9e43101f86e61.png)

# 3.创建TG_bot 机器人
### （1）添加[BotFather](http://t.me/BotFather|)
输入`/start`
再输入`/netbot`
回复你想要的bot名字， 比如: `testbot`
再回复你想要的bot 用户名，必须以`bot`结尾，比如：`test22371_bot`，后面你会得到： `https:t.me/test22371_bot`的链接
记住你的`api`，后面会用到

### （2）配置`BOT`
输入 `/mybots`
选择刚才创建的`test22371_bot`
选择BOT_setting - Domain - 输入要给`remnawave-telegram-sub-mini-app`使用的反代域名，，比如`tgbot.123.com`
选择BOT_setting - Configure Mini app - Enable Mini app - 输入和上面一样的域名,但是要https开头：`https://tgbot.123.com`
选择BOT_setting - Menu Button - Configure Menu Button - 输入`https://tgbot.123.com` - 再输入 "订阅页面" ，就可以得到这样的bot页面了
![a4e13cc2-363d-4078-95da-6485aad80bd0.png](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/a4c597a6303d3698917d3b91af5e8c7ebb809b1d.png)


# 4.部署remnawave-telegram-sub-mini-app

### (1)创建目录

`mkdir /opt/remnawave-telegram-sub-mini-app && cd /opt/remnawave-telegram-sub-mini-app`

### (2)下载环境变量文件

`curl -o .env https://raw.githubusercontent.com/maposia/remnawave-telegram-mini-bot/refs/heads/main/.env.example`

### (3)配置环境变量

`nano .env`

|变量名|值|描述|
|-|-|-|
|`REMNAWAVE_PANEL_URL`|`http://remnawave:3000` 或者 `https://panel.example.com`|同一台机器下使用`http://remnawave:3000`就可以了|
|`REMNAWAVE_TOKEN`|布尔值|在面板设置中创建`API Tokens`|
|`BUY_LINK`||这里我们不填|
|`CRYPTO_LINK`|-|允许使用加密链接（目前支持 Happ 应用）。目前基本没人用，可以不填|
|`AUTH_API_KEY`|-|如果你使用“带安全保护的 Caddy”或 TinyAuth 来获取 Nginx 插件，你可以在这里放置 X-Api-Key，该密钥将应用于 Remnawave 面板的请求。（可以不填）|
|`TELEGRAM_BOT_TOKEN`|布尔值|填写你在[BotFather](http://t.me/BotFather|)得到的`api`
|`FORCE_SNOWFLAKES`|`true`或者`不填`|是否下雪特效，可以为空|

## PS：注意bug！！
目前版本启动过容器后修改以上任何参数都是无效的，必须删除容器重新启动一个。

### (4)创建docker-compose.yml 文件

`nano docker-compose.yml`

以下是面板与小程序同机器的 compose 示例
```
services:
  remnawave-mini-app:
    image: ghcr.io/maposia/remnawave-telegram-sub-mini-app:latest
    container_name: remnawave-telegram-mini-app
    hostname: remnawave-telegram-mini-app
    env_file:
      - .env
    restart: always
    ports:
      - "127.0.0.1:3020:3020"
    networks:
      - remnawave-network

networks:
  remnawave-network:
    external: true

```

### (5)启动docker-compose.yml 
`docker compose up -d && docker compose logs -f`

### (6)Mini 应用现在已经在 http://127.0.0.1:3020 上运行了
现在可以反代了，注意使用的域名和前面一样 