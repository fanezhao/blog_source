---
title: Hexo（一）搭建个人博客
date: 2020-04-13 23:28:08
banner: 
tags:
 - Hexo
---

## 基础环境

- 本地环境：macOS Catalina 10.15.4
  - Nodejs
  - Git
- 个人服务器：华为云 CentOS Linux release 7.7.1908 (Core)
  - Docker
  - Nginx
  - Git

---

## 本地环境搭建

**一、安装Node和Git**

1、Macos 安装Nodejs，到[Nodejs官网](http://nodejs.cn/)下载安装即可，这里省略具体步骤。

2、Macos 安装Git，到[Git官网](https://git-scm.com/)下载安装即可，这里省略具体步骤。

3、安装部署插件：hexo-deployer-heroku

```shell
$ npm install hexo-deployer-heroku --save
```

## 服务器环境搭建

**一、安装Docker 和 Git**

1、Centos 安装Docker，根据[Docker官网](https://www.docker.com/)提供方式安装即可，这里省略具体步骤。

2、Centos 安装Git，根据[Git官网](https://git-scm.com/)提供方式安装即可，这里省略具体步骤。

**二、配置Git Hooks**

1、创建博客静态资源根目录，博客的静态资源就放在这里，我这里叫blogroot。

```shell
$ mkdir blogroot
```

2、创建Git 裸库

```shell
$ mkdir ~/blog.git && cd ~/blog.git
$ git init --bare
```

3、添加Hooks脚本

```shell
$ vim hooks/post-receive

#!/bin/bash
git --work-tree=/home/zmoyi/blogroot --git-dir=/home/zmoyi/blog.git checkout -f 
```

4、脚本添加执行权限

```shell
$ chmod +x hooks/post-receive
```

**三、安装Nginx**

这里我们使用Docker来安装Nginx，也可以使用Centos直接安装Ngixn。

1、拉取Nginx镜像，默认版本是官方版本。

```shell
$ docker pull nginx
```

2、自定义Nginx配置，在启动容器的时候关联。

nginx.conf

```
#运行nginx的用户
user  nginx;
#启动进程设置成和CPU数量相等
worker_processes  1;

#全局错误日志及PID文件的位置
error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

#工作模式及连接数上限
events {
        #单个后台work进程最大并发数设置为1024
    worker_connections  1024;
}

http {
    #设定mime类型
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    #设定日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $request_time $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #设置连接超时的事件
    keepalive_timeout  65;

    #开启GZIP压缩
    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

default.conf

```conf
server {
    listen       80;
    server_name  localhost;	# 这里是直接请求本机的，如果有域名，可以映射成自己的域名。

    #charset koi8-r;
    #access_log  /var/log/nginx/log/host.access.log  main;

    location = / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    location / {
        root   /usr/share/nginx/html;
        try_files $uri $uri/ =404;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

3、启动容器

```shell
$ docker run --detach \
        --name nginx-blog \	# 容器名我定义为nginx-blog
        -p 80:80 \
        -v /etc/localtime:/etc/localtime:ro\
        -v /home/zmoyi/blogroot:/usr/share/nginx/html:rw\	# 注意：这里要把blogroot映射为nginx的静态资源目录
        -v /home/zmoyi/nginx/config/nginx.conf:/etc/nginx/nginx.conf/:rw\
        -v /home/zmoyi/nginx/config/conf.d/default.conf:/etc/nginx/conf.d/default.conf:rw\
        -v /home/zmoyi/nginx/logs:/var/log/nginx/:rw\
        -v /home/zmoyi/nginx/ssl:/ssl/:rw\
        -d nginx
```

4、查询容器状态

{% qnimg nginx_status.png %}

在 STATUS 那项如果是Up，就说明启动成功了。

## 初始化博客

**一、安装Hexo**

1、本地Nodejs安装成功之后，即可使用npm安装Hexo，执行安装命令，等待安装成功即可。

```shell
$ npm install -g hexo-cli
```

2、Hexo安装成功之后，在本地初始化Hexo后，进入到目标文件夹。

```shell
$ hexo init myblog
$ cd myblog
```

**二、安装Hexo主题**

[Hexo主题网站](https://hexo.io/themes/)总有一款适合你。挑选自己喜欢的主题安装即可，我使用的是**anodyne**：

```shell
$ git clone https://github.com/klugjo/hexo-theme-anodyne themes/anodyne
```

clone 完成之后，就可以在 themes 下面看到主题的文件夹。具体主题的使用方式，可以参考主题的使用教程。

**三、本地预览博客**

1、生成静态文件：

```shell
$ hexo generate	# 可以简写成 hexo g
```

2、启动服务器:

```shell
$ hexo sever # 可以简写成 hexo s
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```

默认情况下，访问网址为： `http://localhost:4000/`。这样就可以预览博客了。

**四、部署到GitHub（可选）**

> 如果没有个人服务器，可使用GitHub Pages部署。

1、在自己的GitHub创建个人仓库，命名为：

```shell
github_username.github.io	# github_username 是github用户名
```

2、配置Hexo部署方式：

```yaml
deploy:
  type: git	# 版本控制
  repo: https://github.com/github_username/github_username.github.io.git	# 仓库地址
  branch: master # 分支
  message: # 提交信息，默认值：Site updated: {{ now('YYYY-MM-DD HH:mm:ss') }})
```

3、一键部署：

```shell
$ hexo deploy # 可以简写成hexo d
```

部署成功没有报错的情况下，在GitHub仓库中就可以看到生成的静态文件了。

然后，浏览器访问：https://github_username.github.io/，就可以访问博客了。

**五、部署到个人服务器**

1、修改Hexo部署方式

```yaml
deploy:
  type: git	# 版本控制
  repo: zmoyi@122.112.*.*:/home/zmoyi/blog.git	# 仓库地址
  branch: master # 分支
  message: # 提交信息，默认值：Site updated: {{ now('YYYY-MM-DD HH:mm:ss') }})
```

## 引用链接

- [Hexo官网](https://hexo.io/zh-cn/)