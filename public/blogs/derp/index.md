Tailscale 在网络直连时，默认使用 UDP 协议进行通信。然而，在国内复杂的网络环境下，UDP 包很容易受到运营商的QOS策略影响，导致连接速度慢、延迟高等问题。当通过DERP服务器进行中转时，Tailscale 会切换为TCP协议。

Tailscale官方给出了自建Derp的教程 [custom derp servers](https://tailscale.com/kb/1118/custom-derp-servers)，搭配200M轻量云食用非常丝滑。

但是，当 Tailscale 检测到通过UDP打洞可以直连时，即使直连的性能不佳（直连的网络体验不如经过BGP优化后的 DERP 中转），它也会强制优先使用直连。

## 原理

通过查阅 Tailscale 的代码仓库，可以发现官方预留了一个用于调试的环境变量 `TS_DEBUG_ALWAYS_USE_DERP`。将其设置为 `true`，即可强制 Tailscale 优先使用 DERP 中转，不再尝试 UDP 直连。[代码链接](https://github.com/tailscale/tailscale/blob/b32a01b2dc44986ce83d2dd091f53c31e9a25391/wgengine/magicsock/debugknobs.go#L37)

## 重要提醒

截至 Tailscale 1.84.2 版本，由于 [bug](https://github.com/tailscale/tailscale/issues/16052) ，如果同一台 Linux 服务器既作为 DERP 中转服务器，又运行 Tailscale 客户端并启用强制中转，那么该客户端自身的 Tailscale 会出现掉线问题。
​**请勿在 DERP 服务器上启用强制中转**​，否则会导致其自身的 Tailscale 服务中断。

## 在 Windows 端强制启用/禁用 DERP 中转

### 启用强制 DERP 中转

以管理员权限打开Powershell执行下列命令开启强制中转

```bash
$reg = "HKLM:\SYSTEM\CurrentControlSet\Services\Tailscale"
$val = "TS_DEBUG_ALWAYS_USE_DERP=true"
if (Get-ItemProperty -Path $reg -Name Environment -ErrorAction SilentlyContinue) {
  $cur = Get-ItemPropertyValue -Path $reg -Name Environment
  if ($cur -notcontains $val) {
    Set-ItemProperty -Path $reg -Name Environment -Value ($cur + $val)
  }
} else {
  New-ItemProperty -Path $reg -Name Environment -Value $val -PropertyType MultiString
}
Restart-Service Tailscale -Force
```

### 禁用强制 DERP 中转

执行以下脚本关闭强制中转（这个脚本的Restart-Service时间可能会比较长，耐心等待即可）

```perl
$reg = "HKLM:\SYSTEM\CurrentControlSet\Services\Tailscale"
$val = "TS_DEBUG_ALWAYS_USE_DERP=true"
if (Get-ItemProperty -Path $reg -Name Environment -ErrorAction SilentlyContinue) {
  $cur = Get-ItemPropertyValue -Path $reg -Name Environment
  $new = $cur | Where-Object { $_ -ne $val }
  if ($new.Count -eq 0) {
    Remove-ItemProperty -Path $reg -Name Environment
  } else {
    Set-ItemProperty -Path $reg -Name Environment -Value $new
  }
}
Restart-Service Tailscale -Force
```

## 在 Linux 端强制启用/禁用 DERP 中转

### 启用强制 DERP 中转

sudo权限执行以下脚本开启强制中转

```bash
sudo mkdir -p /etc/systemd/system/tailscaled.service.d
echo -e "[Service]\nEnvironment=TS_DEBUG_ALWAYS_USE_DERP=true" | sudo tee /etc/systemd/system/tailscaled.service.d/override.conf
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart tailscaled
```

### 禁用强制 DERP 中转

执行以下脚本关闭强制中转（脚本执行时间可能比较久）

```bash
sudo rm -f /etc/systemd/system/tailscaled.service.d/override.conf
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart tailscaled
```

本文转载：https://linux.do/t/topic/752216
