# 展示界面
![i/2026/01/02/yks5kx-zjrp.png](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2026/01/02/yks5kx-zjrp.png)
![i/2026/01/02/ykz22r-he0n.gif](https://r2.lyccc.cc/i/2026/01/02/ymc5cg-593b.gif)

# 平台部署方案
在开始部署前，请确保您已经拥有 Cloudflare 和 GitHub 账号。

## 1.Fork [此项目](https://github.com/YYsuni/2025-blog-public)

## 2.登录 Cloudflare 后台，点击左侧的 Compute > Workers，在右侧点击 Pages，然后点击 “Import an existing Git repository” 旁边的 Get Started。 
![image](https://cdn.canjie.org/AgAD2iQAAm3NAVU.webp)

## 3.在弹出的界面中点击 Connect GitHub。 
![image](https://cdn.canjie.org/AgAD0yQAAm3NAVU.webp)

## 4.如果您尚未登录 GitHub，请先完成登录。

登录后，在授权界面中找到 Repository access，选择 Only select repositories，然后点击 Select repositories，勾选刚刚 Fork 的项目，点击 Save 保存授权。授权成功后系统会自动返回 Cloudflare。 
![image](https://cdn.canjie.org/AgAD0SQAAm3NAVU.webp)


## 5.返回 Cloudflare 后，在 “Select a repository” 中选择*刚授权的仓库*，点击激活后的 Begin setup 继续。 
![image](https://cdn.canjie.org/AgAD1SQAAm3NAVU.webp)

## 6.设置构建参数：

- Project name：填写任意项目名称
- Build command：`pnpm run build`
- Build output directory：`/`
然后点击 Save and Deploy，Cloudflare 将自动构建并部署项目。 

## 6.创建 Github App 链接仓库

在 github 个人设置里面，找到最下面的 Developer Settings ，点击进入

![](https://www.yysuni.com/blogs/readme/0abb3b592cbedad6.png)

进入开发者页面，点击 **New Github App**

*GitHub App name* 和 *Homepage URL* , 输入什么都不影响。Webhook 也关闭，不需要。

![](https://www.yysuni.com/blogs/readme/71dcd9cf8ec967c0.png)

只需要注意设置一个仓库 write 权限，其它不用。

![](https://www.yysuni.com/blogs/readme/2be290016e56cd34.png)

点击创建，谁能安装这个仓库这个选择无所谓。直接创建。

![](https://www.yysuni.com/blogs/readme/aa002e6805ab2d65.png)

## 7.创建密钥

创建好 Github App 后会提示必须创建一个 **Private Key**，直接创建，会自动下载（不见了也不要紧，后面自己再创建再下载就行）。页面上有个 **App ID** 需要复制一下
![i/2026/01/02/ykkheg-fm7t.png](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2026/01/02/ykkheg-fm7t.png)
再切换到安装页面

![](https://www.yysuni.com/blogs/readme/c122b1585bb7a46a.png)

这里一定要只**授权当前项目**。

![](https://www.yysuni.com/blogs/readme/2cf1cee3b04326f1.png)

点击安装，就完成了 Github App 管理该仓库的权限设置了。下一步就是让前端知道推送那个项目，就是最开始提到的环境变量。直接改仓库文件 `src/consts.ts` 。

- OWNER: process.env.NEXT_PUBLIC_GITHUB_OWNER || '替换1',`改为你的github id名，比如你主页是https://github.com/woshidashuaibi，那么填写woshidashuaibi`
- REPO: process.env.NEXT_PUBLIC_GITHUB_REPO || '2025-blog-public',`填入你命名的项目名称，如果没有修改就是这个`
- BRANCH: process.env.NEXT_PUBLIC_GITHUB_BRANCH || 'main',
- APP_ID: process.env.NEXT_PUBLIC_GITHUB_APP_ID || '2567830',`app id ，第6步有些`
- ENCRYPT_KEY: process.env.NEXT_PUBLIC_GITHUB_ENCRYPT_KEY || 'wudishiduomejimo',

![](https://www.yysuni.com/blogs/readme/c5a049d737848abf.png)

设置完成后，Cloudflare会自动部署，让环境变量生效。
