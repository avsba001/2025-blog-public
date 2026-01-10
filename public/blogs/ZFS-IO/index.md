# 1.创建规则文件 `/etc/udev/rules.d/60-io-scheduler.rules`
```
ACTION=="add|change", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"
ACTION=="add|change", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="mq-deadline"
```
# 2.应用规则并重启：
```
udevadm control --reload && udevadm trigger
reboot
```
# 3.重启后检查调度器是否生效：
```
cat /sys/block/sdX/queue/scheduler
```