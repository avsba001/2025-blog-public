# Fail2ban + UFW 自动封禁 SSH 探测与爆破 IP (Debian 12 实战)
---
# 2025/09/21 更新
有人私信说屏蔽太多IP有性能问题，我用ipset优化了下，上万IP封禁都不会影响上网
1. 按教程完成所有操作。
2. `apt isntall ipset` 安装ipset包
3. 修改 `/etc/fail2ban/jail.local`
```
#DEFAULT-START
[DEFAULT]
bantime  = 12h
findtime = 10m
maxretry = 5
banaction = iptables-ipset-proto4
action    = %(action_mwl)s
ignoreip  = 127.0.0.1/8
#DEFAULT-END

[sshd]
enabled  = true
filter   = sshd
port     = 22222
maxretry = 2
findtime = 10m
bantime  = 8d
logpath  = /var/log/auth.log

[sshd-disconnect]
enabled  = true
filter   = sshd-disconnect
logpath  = /var/log/auth.log
findtime = 10m
maxretry = 1
bantime  = 8d
```
4. `sudo systemctl restart fail2ban` 重启fail2ban



---
# 2025/08/24 10:54更新
看评论很多朋友说改高位端口和密钥就够了，我在此说明下：
高位端口和密钥只是让别人爆破不到root权限，但是禁止不了爆破和探测这个行为
## 而脚本不仅是普通教程的ssh爆破封禁，同时把探测ssh端口的情况也给封禁了！
## 也就是说
## 看都不准看一眼！ 

---
# 前言
本教程在 **Debian 12** 上测试，使用 `rsyslog`、`ufw` 和 `fail2ban` 作为主要软件。  

### 先说结论：
1. 根据测试推论：  
   - 如果你**不封禁**这些探测 IP 或恶意爆破 SSH 密码的 IP，每个 IP 每天大概会尝试 **4-5 次**。  
   - 如果你**封禁**了它们，这些攻击源会在短时间内把你的 IP 加入黑名单，**不再尝试爆破**。  

2. 实际效果：  
   - 在我上线这个脚本的第一天，`rn` 的 `dc03` 机器就封禁了 **500+ IP**（已修改 SSH 默认端口的情况下）。  
   - 目前每天大约只有 **50 个 IP** 被封禁。  

### 成果展示：
![封禁效果截图](https://avif2.lyccc.cc/https://r2.lyccc.cc/i/2025/08/24/liqki-m4.jpg)

---

## 教程步骤

### 1. 安装必要的软件包
```bash
apt update && apt install rsyslog ufw fail2ban -y
```
### 2. 新建 Fail2ban 过滤器
```bash
nano /etc/fail2ban/filter.d/sshd-disconnect.conf
```
写入以下内容：
```
[Definition]
# 日志格式为 ISO8601 时间戳，后跟 'sshd[PID]: ...'
# 匹配常见爆破行为：
failregex = ^.+sshd\[\d+\]:\s+Disconnected from authenticating user root <HOST> port \d+ \[preauth\]$
            ^.+sshd\[\d+\]:\s+Received disconnect from <HOST> port \d+:11: Bye Bye \[preauth\]$
            ^.+sshd\[\d+\]:\s+Invalid user .* from <HOST> port \d+.*$
            ^.+sshd\[\d+\]:\s+Disconnected from invalid user .* <HOST> port \d+ \[preauth\]$

# 防止误伤已成功登录的记录
ignoreregex = ^.+sshd\[\d+\]:\s+Accepted .+ from <HOST> port \d+ .*$

# 可选规则：激进封禁（谨慎开启）
#           ^.+sshd\[\d+\]:\s+Connection closed by <HOST> port \d+ \[preauth\]$
```
### 3. 修改 Fail2ban Jail 配置
编辑 `jail.local`：
`nano /etc/fail2ban/jail.local`
```
[sshd-disconnect]
enabled  = true
filter   = sshd-disconnect
logpath  = /var/log/auth.log
maxretry = 2
findtime = 600
bantime  = 24h
ignoreip = 127.0.0.1/8
banaction = ufw
```

完整`Jail` 配置展示
```
#DEFAULT-START
[DEFAULT]
bantime = 600
findtime = 300
maxretry = 5
banaction = iptables-allports
action = %(action_mwl)s
#DEFAULT-END

[sshd]
ignoreip = 127.0.0.1/8
enabled = true
filter = sshd
#自己的ssh端口
port = xxxx  
maxretry = 2
findtime = 600s
bantime = 80000s
banaction = ufw
action = %(action_mwl)s
logpath = /var/log/auth.log

[sshd-disconnect]
enabled = true
filter  = sshd-disconnect
logpath = /var/log/auth.log
maxretry = 2
findtime = 600
bantime  = 24h
ignoreip  = 127.0.0.1/8
banaction = ufw
```
### 4. 启用并检查 Fail2ban
```systemctl enable fail2ban --now
systemctl status fail2ban
```
查看是否生效：
```
fail2ban-client status sshd-disconnect
```
### 总结

通过 Fail2ban + UFW，可以高效屏蔽 SSH 探测和爆破行为。

经过一段时间后，恶意 IP 会停止对你服务器的尝试，大幅减少安全威胁和日志垃圾。