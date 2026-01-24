# 1.配置git账户
```
git config --global user.name "用户名"
git config --global user.email "邮箱"
```
![1](https://picx.zhimg.com/v2-4185f37f3a82db89f0806a39aca85481_1440w.jpg)

执行完后，再执行 `git config --global --list`  查看设置好的用户名和账户
![2](https://pic3.zhimg.com/v2-cd3d83503e7aaba8d6ce9bc7fe14977e_1440w.jpg)

# 2.关联github账户
## 配置SSH KEY
```
ssh-keygen -t rsa -C "xxx@xx.com"
```
一路回车

打开文件查看，路径为 **C:\Users\用户名\.ssh**， 记住这里的 **pub** 文件

打开github-setting-ssh and GPG key 。

添加一个 **NEW SSH KEY**

![](https://pic3.zhimg.com/v2-750bd7001c532b22e8e2c5c636a10562_1440w.jpg)

## 检验ssh keys是否添加成功
```
ssh -T git@github.com 
```
# 关联完成，下面开始git 仓库
---

# 3.配置本地代理

由于我们只针对github 。临时设置代理为
```
git config --global http.https://github.com.proxy http://127.0.0.1:XXXX
```

# 4. clone到本地
```
git clone https://github.com/你的用户名/仓库名.git
```
# 5. 添加上游仓库（只做一次）
```
git remote add upstream https://github.com/原作者/项目名.git
```
# 6. 获取上游最新代码（不会改你任何东西）
```
git fetch upstream
```
这一步 **只是下载**，很安全。

# 7.查询差异文件

去除`--name-only`可以查看文件内容差异
```
git diff --name-only upstream/main    
```

# 8.同步指定文件或文件夹
**同步 单个文件**
```
git checkout upstream/main -- src/config.js
```
**同步 整个目录**
```
git checkout upstream/main -- src/utils/
```

# 9. 提交这次“选择性同步”
```
git status                               # 查看当前目录文件状态
git add src/config.js              # 将文件加入暂存区，`add .` 为提交当前目录
git commit -m "Sync upstream"   # 合并暂存区至仓库
```

