---
title: 踩坑DN42-p2-peer!
date: 2021-12-13 01:45:56
tags: tech
categories: note
---

这篇文章可能会存在不精确、错误、过时等问题。遇到时请务必留言给喵喵修改！！

在上一集中，俺们在DN42上注册了一个ASN，并申请到了IP地址池。那接下来，就该开始和别人建立连接了。
而这，就叫做：
# PEER!

## before

你可能需要：
更新一下系统和软件
`apt update && apt upgrade`

关闭**所有的**防火墙，并打开路由转发，关闭rp_filter
```
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.forwarding=1" >> /etc/sysctl.conf
echo "net.ipv6.conf.all.forwarding=1" >> /etc/sysctl.conf
sysctl -p

echo "net.ipv4.conf.default.rp_filter=2" >> /etc/sysctl.conf
echo "net.ipv4.conf.all.rp_filter=2" >> /etc/sysctl.conf
sysctl -p
```
（及其建议自行dd一个镜像进去，除非你想感受某些vps厂商给你准备的“惊喜”）

## step1:建立VPN隧道
一般来说，在现实中比较大的ISP之间，会通过网线/光纤直链等方式，将双方的网卡建立一个链路。
当然在DN42的网络中……这么做好像有亿点点的不现实？（哪怕是一个城的，谁会真的在现实里从东到西接一根光纤就为了把两台电脑连起来啊喂！）
于是，我们便不得不使用VPN来在现实的网络上凭空架起一层，模拟出各个网卡间的物理连接。

废话不多说，上配置！
**老规矩，之后的FOO或者其它的命名请自行替换为你觉得舒服的格式**
*以下是名字的部分我会打上'\[','\]'*

<需要注意的是，喵喵暂时全网使用wireguard进行点对点VPN连接（体会下来感觉也是最广泛的一个？）>

来我们先安装以下Wireguard:（你可能需要添加 []-backports 源）
`apt install wireguard-tools wireguard-dkms`

```
[Interface]
PrivateKey = [YOUR WG PRIVATE KEY]
ListenPort = [THE PORT YOU WANT OTHER SENDING DATA]
Table = off
PostUp = ip addr add [YOUR IPV6 LINK LOCAL] dev %i
PostUp = ip addr add [YOUR IPV6] dev %i
PostUp = ip addr add [YOUR DN42 IPV4] peer [THE DN42 IPV4 FROM THE MACHINE YOU WANT TO PEER] dev %i
PostUp = sysctl -w net.ipv6.conf.%i.autoconf=0

[Peer]
PublicKey = [THE WG PUBLIC KEY FROM THE MACHINE YOU WANT TO PEER]
Endpoint = [ENDPOINT]:[PORT]
AllowedIPs = 10.0.0.0/8, 172.20.0.0/14, 172.31.0.0/16, fd00::/8, fe00::/8
```

看不懂？来慢慢的解释一下。
首先，让我们思考一下：怎么确定你想连接的机器就是那一台，而非一个中间人的呢？
答案和https验证机制类似：一个公私密钥对。
于是，首先我们运行`wg genkey | tee privatekey | wg pubkey > publickey`产生一对公私钥对，然后**千万不要学某只喵喵一样把私钥给泄露出去无数遍了！**
那这也就解决了PrivateKey和PublicKey后面该填什么了：你的私钥和*对方的*公钥

接下来是你期望监听的端口。一般而言你可以选择它ASN的后五位（直到你遇到了一个另一个网络的（X
按自己喜欢的规律来就好啦~（记得不要把自己绕进去哦！）

Link Local的话是要在fe80::下面的哦

然后你就可以`wg-quick up [你刚刚给这个配置文件取的名称]`，再ping一下对方的DN42 IP，塔哒~通了w~

## step2:建立BGP
 
（其实理论上说，应该先建立好IGP(内部的网络连接)，再建立BGP，但是大部分人最开始好像都只有一台机子来着？）

喵喵及其建议使用Bird2（而且使用[]-backport的高版本，会有新功能哦~）

来，先不管什么意思从Wiki上把这个配置复制进去：[https://wiki.dn42.us/howto/Bird2](https://wiki.dn42.us/howto/Bird2)

然后在/etc/bird/peer下面建立一个配置文件（一样，名字按你的喜好来）

```
protocol bgp [V4 NAME] from dnpeers {
	neighbor [IP OF THE OTHER MACHINE] as [ASN];
	direct;
	ipv6 {
		import none;
		export none;
	};
};

protocol bgp [V6 NAME] from dnpeers {
	neighbor [LING LOCAL OF THE OTHER MACHINE] % '[NAME]' as [ASN];
	direct;
	ipv4 {
		import none;
		export none;
	};
};
```

需要注意的是，这里给出的配置**既没有mBGP（多个协议走同一个通道），也没有Extended Next Hop**（前面的蛆以后再来探索吧~~）
但毕竟能用，而且好用

然后，bird提供了一个很好用的工具：birdc
你可以通过`birdc c`来重载配置，也可以通过`birdc s p`来查看连接情况
好啦，现在你所知道的东西已经足以应付大部分情况啦~开始你的BGP之旅吧探险者！（xxx

---
未完待续...