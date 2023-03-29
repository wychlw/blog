---
title: 搭建一个KMS服务器
date: 2021-10-13 15:34:38
tags: tech
categories: note
---

# 0x55aa 为什么
因为好玩！（x
当然，市面上已经有着各种的kms激活器。可，每一个激活器都会多多少少的报毒
虽然我们对于那些激活器的作者十分信任，但是……谁知道有木有谁在里面真的藏了点什么呢？
嘛……所以，最放心的方式当然是自己来啦~

# 0x00 什么是KMS？
[Microsoft官方Q&A](https://docs.microsoft.com/en-us/licensing/products-keys-faq#what-is-volume-activation)
[Wikipedia的介绍](https://zh.wikipedia.org/zh-hans/KMS)
KMS，全称Key Management Service，是Windows批量激活的方式。
众所周知，当使用个人版的Windows时，会让你输入一个激活密钥
但是！当到了企业、学校中的时候……总不可能买一堆堆密钥回来一个一个手动激活吧
于是，这个时候在企业、学校内网之中便可以lei部署一台服务器用来专门激活这些批量版的Windows，而这个激活使用的服务就是KMS

# 0x01 搭建

你需要：
- 一台非本机的电脑/开一个虚拟机（Windows禁止了激活服务器和被激活机器为同一台机器
- 没了。（就介么简单！

首先下载一下这个模拟KMS服务器的东西
[vlmcsd](https://github.com/Wind4/vlmcsd)
make一下
然后运行它
`vlmcsd -L 0.0.0.0:1688`
没了。
（就这么简单）

# 0x02 使用

如果 你的Windows是零售版：
    你要先用下面的代码将它转换为批量激活的版本
```powershell
cd C:\Windows\System32
cscript slmgr.vbs /skms kms.wcysite.com
cscript slmgr.vbs /ato
cscript slmgr.vbs /xpr
```

接下来你可以：
1. 使用powershell/cmd激活（记得开管理员鸭！）
```powershell
cd C:\Windows\System32
cscript slmgr.vbs /ipk W269N-WFGWX-YVC9B-4J6C9-T83GX
```
2. 直接在设置中输入密钥激活

## 自动启动？
你可以添加一个systemd
```
[Unit]
Description=Windows kms server
After=network.target

[Service]
ExecStart=/usr/local/bin/vlmcsd -L 0.0.0.0:1688
Type=forking
ExecStop=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=30s

[Install]
WantedBy=multi-user.target
```

## 密钥从哪来？
[微软官方](https://docs.microsoft.com/en-us/windows-server/get-started/kmsclientkeys)其实给出了密钥，自己进去按照你的版本复制吧~

唔姆……大概……就介样？