---
title: 一种利用机场搭建wireguard VPN的方法
date: 2022-03-07 15:39:10
tags: tech
categories: note
---

# 原因
- 机场可以更大程度上的保证连接的稳定性（不会被乱掐）
- v2ray中支持的socks、websocket、http等代理方式均是5层的代理，而wireguard是一个三层的代理。在搭建一些必须要在一个子网内的服务时，三层代理便是一个必不可少的选项了。

# 声明
**我使用这种方式的原因只是为了使得跨境的vps位于同一子网中通信。请在使用任何方法时注意法律法规及公序良俗，对于任意基于此方法的行为请自行斟酌。**

# 前期准备
## 安装必要软件
*以下均以Debian-bullseye为例*
首先，在需要连接的两台机器上安装wireguard与v2ray
（v2ray仅需要在主动连接到机场的机器上进行安装）
```bash
apt update
apt upgrade
apt install wireguard-tools wireguard-dkms
apt install v2ray
```
（v2ray已经被Debian官方收录进了源内。若您采用的系统没有收录，请参考[v2ray官方的教程](https://v2ray.com/chapter_00/install.html)）

## 确定机场的配置
对于大多数的机场，都并不会给v2ray-core的配置文件。对此的解决方式是：用v2rayN来转换
1. 打开v2rayN并配置订阅源
![配置订阅源](https://js-d.wcysite.com/gh/wychlw/img@main//img/20220307160215.png)
2. 在你想使用的节点上右键，并选择*导出为客户端配置*
![导出配置](https://js-d.wcysite.com/gh/wychlw/img@main//img/20220307160528.png)
3. 复制配置文件中，outbounds下**tag为proxy的部分**
![配置文件](https://js-d.wcysite.com/gh/wychlw/img@main//img/20220307160950.png)

# 进行配置
## 编写v2ray-core的文件
**仅需要对主动连接到机场的机器进行编写**
*v2ray的默认配置在`/etc/v2ray/config.json`*

直接复制以下配置即可。记得将本机端口、服务器地址、端口和机场配置进行替换。
（解释在配置之后）

```json
{
    "log": {
        "access": "none",
        "error": "/var/log/v2ray/error.log",
        "loglevel": "warning"
    },
    "dns": {
        "hosts": {},
        "servers": [
            "8.8.8.8",
            "8.8.4.4",
            "localhost"
        ]
    },
    "inbounds": [
        {
            "port": 本机的端口,
            "protocol": "dokodemo-door",
            "settings": {
                "address": 目标服务器的地址,
                "port": 目标服务器的端口,
                "network": "tcp,udp"
            },
            "tag": "wireguard_inbound_proxy"
        }
    ],
    "outbounds": [
        你复制的机场配置
    ],
    "routing": {
        "rules": [
            {
                "type": "field",
                "inboundTag": [
                    "wireguard_inbound_proxy"
                ],
                "outboundTag": "proxy"
            }
        ]
    }
}
```

这里其实是使用了v2ray的docodemo-door协议进行了转发。docodemo-door的效果是：将你本机的指定端口，通过代理直接发送到目标服务器的指定端口。也就是说，建立了一个端口映射。

## wireguard配置
以下将主动连接到机场的机器称为A，目标服务器称为B
### A的配置

```conf
[Interface]
PrivateKey = A的私钥
# 以下interface部分请尽情的添加你所需要的功能，这里只给出将两机器连一起的配置
PostUp = ip addr add 虚拟网络中A的ip peer 虚拟网络中B的ip dev %i

[Peer]
PublicKey = B的公钥
Endpoint = 127.0.0.1:A的端口
AllowedIPs = 0.0.0.0/0, ::/0
```

其中，A的端口是上方v2ray配置中指明的本机端口。

#### 为什么endpoint的地址会是127.0.0.1
这里我画一幅图来解释一下：
![解释](https://js-d.wcysite.com/gh/wychlw/img@main//img/20220307163052.png)

### B的配置

```conf
```conf
[Interface]
PrivateKey = B的私钥
# 以下interface部分请尽情的添加你所需要的功能，这里只给出将两机器连一起的配置
PostUp = ip addr add 虚拟网络中B的ip peer 虚拟网络中A的ip dev %i

[Peer]
PublicKey = A的公钥
AllowedIPs = 0.0.0.0/0, ::/0
```

这里注意到并没有配置endpoint。
这是因为B机器并不知道机场服务器的出口在哪（而且哪怕你知道机场也不会让你连的啊kora），所以这里只进行监听，由A主动建立udp连接。

# 启动连接
## A
```bash
systemctl start v2ray
wg-quick up 配置文件名字
```
### 可能出现的错误：
#### v2ray报错：找不到log文件
创建相应文件：`/var/log/v2ray/error.log`
#### v2ray报错：没有权限访问log文件/log文件为只读
删除`/lib/systemd/system/v2ray.service`中dynamicuser=true这一项（我真的不会用呜呜呜）
将log文件权限改为755

## B
```bash
wg-quick up 配置文件名字
```

# end
老实说……挺水的（
所以，接下来你可以往上面再套一层GRE，你就获得了2层代理
或者你可以继续套下去以获得更大的安全裕度
你甚至可以尝试v2ray套娃…………毕竟docodemo-door协议支持所有tcp/udp连接的转发
awa