---
title: 来使用Worker做免费的Email服务器!
date: 2024-03-01 20:16:56
tags: tech
categories: note
---

## 前言

由于现在用的自定域名服务商不给开IMAP/SMTP~~（说的就是你！zoho）~~，而市面上其它的服务商要么需要钱钱，要么只给收信……在这样的条件下，很早就想要自己搭建一个邮件服务器了。但因为自己服务器上奇奇怪怪的问题一直懒得解决，就一直拖着。

而自从Cloudflare推出了它们的邮件路由之后，很多人就开始用它转发邮件。但此时的Email Worker大多数还只停留在只能收信->筛选->转发的阶段。而如果你想用Worker发信呢？对不起！用第三方API吧。

直到不知道什么时候，Cloudflare突然给Email Routing加上了发信的能力（有多突然呢？它自己的API文档都还没更新，types也有问题）。在发现了这一点之后，我就开始思索能否用它的Worker来做一个完整的邮件服务器，包括收发信的能力，而非只是转发到自己的邮箱。

## Email Routing 和 Email Worker

这是Cloudflare不知何时推出出来的服务。本质上就是让Worker支持了SMTP的服务，并添加了MX记录到cloudflare的服务器。

官方支持了它可以实现回信和向前转发的功能，但是都很有限制（如转发必须是你验证的邮箱，也不能随便回信）。

它判断是不是你的Worker处理的方式是收信域是你的Worker配置域。

而发信是它偷偷添加上的功能，可以发送一些Worker的情况。但它自己的blog都还在教怎么用第三方合作API来完成这点所以……emmm这个功能确实好赶啊感觉。

## 一些问题

首先我们需要注意到：虽然Worker支持着很多的协议
![](https://cdn.jsdelivr.net/gh/wychlw/img@main///img/202403012031033.png)

但:
- 它并没有**收信协议**(POP3/IMAP)
- 它的发信协议并不是能给你连接的！（需要目标地址是你的域）

为此，我们不得不自己解决这两个协议。首先我们需要提一些要求：所作出的服务最好能在Windows和Linux上都能使用，最好支持一些常用的客户端。在此我想到了两种方法：
- 使用Microsoft提供的Exchange Server
- 自行搞一个协议和对应的服务出来

对于其它的方法，它们大多并没有很好的客户端适配性。如Windows Mail就只支持各个收发信协议和Exchange Server，因此弄个cron来获取到本地文件夹显然是不太行的。

那使用Exchange Server呢？它固然是很好的：基于HTTP协议意味着你不需要在本机进行任何的修改；同时它也足够泛用到在几乎每个客户端都有支持。但……你确定要自己实现一遍这个协议？？？

因此，我的选择便是：自行在本机与Worker之间构建一套HTTP API，并在本地转换为POP3/SMTP。

## 实现

对应的项目你可以在Github上找到：[Neko Email Worker](https://github.com/wychlw/neko_email_worker)。

### Client Side

在本地，通过Node.js提供了一个小的POP3和SMTP后端来与你本地的客户端通讯。同时，通过Electron+Vue封装了一个小的前端用来配置。
![](https://cdn.jsdelivr.net/gh/wychlw/img@main///img/202403012111333.png)

你可以将它开机自启到后台，然后就可以愉快的使用啦！

它本身是通过一套[RESTful API](https://github.com/wychlw/neko_email_worker/blob/main/protocols.md)来和Worker进行通讯的。

### Worker Side

在Worker上面我们本质只需要做以下几件事：管理用户，为每位用户收发信和管理信件。

用户方面，提供了允许注册用户和管理员（可以随意删改别的用户）的功能。

收信是相对简单的：只需要根据`Recp To`确认用户存在，然后在数据库里插入就行了。

而发信则需要遵守Cloudflare定下的一些规矩：你需要在配置中添加一个`SendMail`字段来表示允许发信的地址。而这带来了一些问题……（见*限制*）

## 限制

就如同前言里面提到的那样……Cloudflare自己都还没有怎么更新它自己的API！例如在本地配置文件里，对应的配置都是没有自己的字段的：
![](https://cdn.jsdelivr.net/gh/wychlw/img@main///img/202403012054228.png)

同样的，你自然不可能去在线的通过API更新你这部分的配置……

这导致目前的用户系统还是个半残状态：你能添加一个收信用户，但需要重新构建才能添加一个发信用户。实际上我已经在其中做了尝试在线添加配置，但Cloudflare那是200给的满满的，这部分的配置一点没变（泪）。不过这也意味着：只要它在API里加上支持，这个功能就可以立刻启用！

## End

关于它的使用方法放在Readme里了。由于不太完善不过还没有打包……唉还是后端好写多啦————

## 关于CF的一些小吐槽

这是那个用来在线配置的API…
![](https://cdn.jsdelivr.net/gh/wychlw/img@main///img/202403012119886.png)

如你所见，它要你按`multipart/formdata`上传一个如下的object。我想过是一个长这样的form；写错了其实就是javascript。但是为什么，为什么是一个键叫`setting`，值是下面那一堆json然后stringfy一下呢……？但凡给个例子或者改个形式都行吧……