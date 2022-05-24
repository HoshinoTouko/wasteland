---
title: 卸载腾讯云安全组件
date: 2022-05-24 14:24:38
categories:
- [Network, VPS]
- [Website]
tags:
- VPS
- Website
---

# 前言

今天早上收到一条提醒，用来做梯子的腾讯云机器被提示封禁了。

>尊敬的腾讯云用户，您好！  
您的腾讯云账号（账号ID: *********， 昵称: ******）下的Lighthouse服务存在违规信息，涉嫌违反相关法律法规和政策，101.32.26.12已被限制访问，感谢您理解和支持！  
违规类型：存在通过技术手段使其成为跨境访问节点等行为  
违规URL：N/A  
违规域名：N/A  
违规标识：101.32.26.12

这让我感觉很好奇：到底是什么神秘力量探测到我的梯子了。遂怀疑是腾讯云可能存在和阿里云类似的监控用户隐私的类云盾功能，结果真检索到了相关信息。

# 处理腾讯云安全（审查）组件

理论上，现在新创建的所有服务器，包括轻量都预装了这个组件。通过以下命令检查是否被插了后门。

```bash
ps -A | grep agent
```

这时候你会看到

```bash
root@VM-4-9-debian:~# ps -A | grep agent
   1229 ?        00:00:00 sgagent
   1282 ?        00:00:00 barad_agent
   1288 ?        00:00:00 barad_agent
   1289 ?        00:00:00 barad_agent
```

果然是插满了后门啊。

解决方法很简单，参考 hostloc 论坛帖子提供的方法，直接在ssh执行

```bash
sudo -i
systemctl stop tat_agent
systemctl disable tat_agent
/usr/local/qcloud/stargate/admin/uninstall.sh
/usr/local/qcloud/YunJing/uninst.sh
/usr/local/qcloud/monitor/barad/admin/uninstall.sh
rm -f /etc/systemd/system/tat_agent.service
rm -rf /usr/local/qcloud
rm -rf /usr/local/sa
rm -rf /usr/local/agenttools
rm -rf /usr/local/qcloud
process=(sap100 secu-tcs-agent sgagent64 barad_agent agent agentPlugInD pvdriver )
for i in ${process[@]}
do
  for A in $(ps aux | grep $i | grep -v grep | awk '{print $2}')
  do
    kill -9 $A
  done
done
```

即可。最后再使用 `ps -A | grep agent` 检查一次。

# References

- https://hostloc.com/thread-1004087-1-1.html
