---
title: n8n—一款开源信息流工具的自搭建与使用
date: 2021-12-30 07:40:23
tags: tags
categories: note
highlight_shrink: false
---

# 什么是n8n
n8n是一款开源的自动工作流处理工具。
举个例子，现在老师要你按照excel里的邮箱给每个人发送些许不同的邮件（可能名字有区别啊）
或者要求你每次把一个平台上的内容转发到另一个平台上啊
或者同步几个文件的数据啊……
**甚至！在进门之后关灯这种事**
有些时候，这种看起来写个脚本就能重复做的事情超多，但是脚本写起来又会有一点麻烦
你就需要n8n这种图形化的节点编辑工具啦~~
![比如像这样！](https://js-d.wcysite.com/gh/wychlw/img@main//img/20211230075656.png)
（至少它官网是这么写的）
但是在生活中捏，我一般用它来同步博客/twitter/QQ空间/bilibili……上面发布的内容
（比如，你要是在bilibili或者twitter上面看到了这篇文章的链接，那就是n8n推送过去的
（懒是人类进步的阶梯嘛————

# 为什么用n8n

它免费。
你还需要第二个理由吗！！！（xxx

---

那让我们开始搭建吧~~
[这里是官方的doc](https://docs.n8n.io/getting-started/)
//老实说官方的比我写的好多了。为什么不按照官方的文档来一把，然后goto usage呢？

## before

### Docker
当然，你可以把n8n直接搭建在裸机上，或者……还是套层Docker吧，方便以后迁移

以下均以~~某盒装安装介质~~Debian系统为例

那，安装docker的第一步：卸载docker（旧版本啦
`apt-get remove docker docker-engine docker.io containerd runc`

然后安装一点点小工具来添加docker的apt源
`apt-get install ca-certificates curl gnupg lsb-release`

添加源
`curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`

设置stable源
`echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

然后安装它！
```bash
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io
```

## install

接下来会根据环境进行区分，请自行跳转一下哦

### install with security

**这种方式是可以带鉴权安装的，同时建议服务器上使用此种方式**

安装Docker Compose
（是你的内网/其它人无法访问的服务器？没有必要了继续，请直接goto install
```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.27.4/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

设置一个指向你服务器的n8n url

创建一个`docker-composer.yml`，然后把下面这一大段东西粘贴进去
```yaml
version: "3"

services:
  traefik:
    image: "traefik"
    restart: always
    command:
      - "--api=true"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ${DATA_FOLDER}/letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro

  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "127.0.0.1:5678:5678"
    labels:
      - traefik.enable=true
      - traefik.http.routers.n8n.rule=Host(`${SUBDOMAIN}.${DOMAIN_NAME}`)
      - traefik.http.routers.n8n.tls=true
      - traefik.http.routers.n8n.entrypoints=web,websecure
      - traefik.http.routers.n8n.tls.certresolver=mytlschallenge
      - traefik.http.middlewares.n8n.headers.SSLRedirect=true
      - traefik.http.middlewares.n8n.headers.STSSeconds=315360000
      - traefik.http.middlewares.n8n.headers.browserXSSFilter=true
      - traefik.http.middlewares.n8n.headers.contentTypeNosniff=true
      - traefik.http.middlewares.n8n.headers.forceSTSHeader=true
      - traefik.http.middlewares.n8n.headers.SSLHost=${DOMAIN_NAME}
      - traefik.http.middlewares.n8n.headers.STSIncludeSubdomains=true
      - traefik.http.middlewares.n8n.headers.STSPreload=true
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER
      - N8N_BASIC_AUTH_PASSWORD
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    volumes:
      - ${DATA_FOLDER}/.n8n:/home/node/.n8n
```
//如果你想让n8n直接把文件存在某个文件夹下的话，请在volume下加上这样一句话
`- <path you want to save file>:/files`
//如果你不想要鉴权的话，请将environment`N8N_BASIC_AUTH_ACTIVE`设置为false

然后将以下配置粘贴进`.env`里，并**按你的环境修改**
```docker
# Folder where data should be saved
DATA_FOLDER=/root/n8n/

# The top level domain to serve from
DOMAIN_NAME=example.com

# The subdomain to serve from
SUBDOMAIN=n8n

# DOMAIN_NAME and SUBDOMAIN combined decide where n8n will be reachable from
# above example would result in: https://n8n.example.com

# The user name to use for authentication - IMPORTANT ALWAYS CHANGE!
N8N_BASIC_AUTH_USER=user

# The password to use for authentication - IMPORTANT ALWAYS CHANGE!
N8N_BASIC_AUTH_PASSWORD=password

# Optional timezone to set which gets used by Cron-Node by default
# If not set New York time will be used
GENERIC_TIMEZONE=Asia/Shanghai

# The email address to use for the SSL certificate creation
SSL_EMAIL=cat@dog.bird

```

#### 如果你的机器上没有其它的网络应用了
恭喜你！直接运行下面这一条命令，开始你的旅程吧！
`sudo docker-compose up -d`
`sudo docker-compose stop`//如果你想关闭的话

#### 如果有的话！
（比如nginx，apache……）
好吧，由于0.0.0.0:80和0.0.0.0:443被它们监听了，你需要
- 简单点：在上面的docker-compose.yml里换个端口
- 复杂点：反代

首先，我们需要把上面的docker-compose.yml里面`ports:`下改为
```
3080:80
30443:443
```
（当然，你大可不必也用3080和30443

接下来以nginx反代为例：（自行修改！
```nginx
server {
    listen 80;
    server_name  n8n.wcysite.com;
    rewrite ^(.*) https://$server_name$1 permanent;
};

server {

    listen       443 ssl;
    server_name  n8n.wcysite.com;
    location / {
        proxy_pass https://127.0.0.1:30443/;
        proxy_set_header  Host $host;
        proxy_set_header  X-Real-IP  $remote_addr;
        proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
        client_max_body_size 100m;
        client_body_buffer_size 128k;
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        proxy_buffer_size 4k;
        proxy_buffers 4 32k;
        proxy_busy_buffers_size 64k;
        proxy_temp_file_write_size 64k;
        add_header X-Frame-Options SAMEORIGIN;
    }
    ssl_certificate      /etc/letsencrypt/live/n8n.foo.bar/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/n8n.foo.bar/privkey.pem;
    ssl_session_timeout  5m;
    ssl_protocols  SSLv3 TLSv1;
    ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
    ssl_prefer_server_ciphers   on;
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
};
```

接下来一样
`sudo docker-compose up -d`
`sudo docker-compose stop`//如果你想关闭的话

### install on a machine(local or secure)

**这种方法没有任何的鉴权措施！**

有了docker，那安装n8n就是一句话的事了
```
docker run -it --rm --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n n8nio/n8n
```

这句话会在当前的用户文件夹下产生一个.n8n文件夹来存放数据，并会让应用运行在5678端口上

~~（然后你就会发现你的5678端口门户大开x~~
n8n提供了基本的鉴权，但需要按照上面在server上安装的方法安装

## usage
这玩意就是将节点妥妥拽拽来完成工作的，应该不难上手
这里着重讲一下twitter的使用吧

### twitter
n8n连接twitter，本质上是你在twitter上创建了一个你自己的app，然后让n8n连接那个app
那么自然，我们要先：
#### 创建twitter app

首先你需要登录[twitter的开发者页面](https://developer.twitter.com/en/portal/projects-and-apps)，并创建一个app
![创建app](https://js-d.wcysite.com/gh/wychlw/img@main//img/%E6%89%B9%E6%B3%A8%202021-12-14%20000124.png)
然后选择Development（这里是因为我本身就有一个，所以选择不了）
![](https://js-d.wcysite.com/gh/wychlw/img@main//img/20211214000546.png)
取名，然后假如你是李华，你要…………（xxxx
一般来说都会过的啦~~

然后在n8n点击节点![](https://js-d.wcysite.com/gh/wychlw/img@main//img/20211230072615.png)
选这个~![](https://js-d.wcysite.com/gh/wychlw/img@main//img/20211230072733.png)

接下来把key和token填过去就好啦~~
（记得填写callback url哦！

### 一些有用的节点：

#### 当有新feed时：
（请在尾端连上你想去的节点
```json
{
  "nodes": [
    {
      "parameters": {},
      "name": "Start",
      "type": "n8n-nodes-base.start",
      "typeVersion": 1,
      "position": [
        240,
        300
      ]
    },
    {
      "parameters": {
        "url": "https://blog.wcysite.com/rss2.xml"
      },
      "name": "RSS Feed Read",
      "type": "n8n-nodes-base.rssFeedRead",
      "typeVersion": 1,
      "position": [
        480,
        300
      ]
    },
    {
      "parameters": {
        "mode": "jsonToBinary",
        "options": {}
      },
      "name": "Move Binary Data",
      "type": "n8n-nodes-base.moveBinaryData",
      "typeVersion": 1,
      "position": [
        880,
        160
      ]
    },
    {
      "parameters": {
        "fileName": "/files/blog_rss_last_item"
      },
      "name": "Write Binary File",
      "type": "n8n-nodes-base.writeBinaryFile",
      "typeVersion": 1,
      "position": [
        1100,
        160
      ]
    },
    {
      "parameters": {
        "filePath": "/files/blog_rss_last_item"
      },
      "name": "Read Binary File",
      "type": "n8n-nodes-base.readBinaryFile",
      "typeVersion": 1,
      "position": [
        480,
        540
      ]
    },
    {
      "parameters": {
        "options": {}
      },
      "name": "Move Binary Data1",
      "type": "n8n-nodes-base.moveBinaryData",
      "typeVersion": 1,
      "position": [
        660,
        540
      ]
    },
    {
      "parameters": {
        "functionCode": "return [ items[0] ];"
      },
      "name": "Select New",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        660,
        300
      ]
    },
    {
      "parameters": {
        "fileName": "/files/blog_rss_last_item"
      },
      "name": "Clear Old",
      "type": "n8n-nodes-base.writeBinaryFile",
      "typeVersion": 1,
      "position": [
        780,
        820
      ]
    },
    {
      "parameters": {
        "functionCode": "items[0].json.urls = [];\nreturn items;"
      },
      "name": "Function",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        460,
        820
      ]
    },
    {
      "parameters": {
        "mode": "jsonToBinary",
        "options": {}
      },
      "name": "JSON to Data1",
      "type": "n8n-nodes-base.moveBinaryData",
      "typeVersion": 1,
      "position": [
        620,
        820
      ]
    },
    {
      "parameters": {},
      "name": "Merge",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 1,
      "position": [
        880,
        420
      ]
    },
    {
      "parameters": {
        "functionCode": "// Code here will run only once, no matter how many input items there are.\n// More info and help: https://docs.n8n.io/nodes/n8n-nodes-base.function\n\n// Loop over inputs and add a new field called 'myNewField' to the JSON of each one\n\nif (items[0].json.link!=null&&items[1].json.link!=null&&items[0].json.link==items[1].json.link)\n{\n  return [];\n}\n\nreturn [items[0]];"
      },
      "name": "Function1",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [
        1100,
        420
      ]
    }
  ],
  "connections": {
    "Start": {
      "main": [
        [
          {
            "node": "RSS Feed Read",
            "type": "main",
            "index": 0
          },
          {
            "node": "Read Binary File",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "RSS Feed Read": {
      "main": [
        [
          {
            "node": "Select New",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Move Binary Data": {
      "main": [
        [
          {
            "node": "Write Binary File",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Read Binary File": {
      "main": [
        [
          {
            "node": "Move Binary Data1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Move Binary Data1": {
      "main": [
        [
          {
            "node": "Merge",
            "type": "main",
            "index": 1
          }
        ]
      ]
    },
    "Select New": {
      "main": [
        [
          {
            "node": "Move Binary Data",
            "type": "main",
            "index": 0
          },
          {
            "node": "Merge",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Function": {
      "main": [
        [
          {
            "node": "JSON to Data1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "JSON to Data1": {
      "main": [
        [
          {
            "node": "Clear Old",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Merge": {
      "main": [
        [
          {
            "node": "Function1",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```
