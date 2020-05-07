---
title: Hexo（二）整合七牛云图床
date: 2020-04-14 28:47:50
banner: 
tags:
 - Hexo
---

## 注册七牛云

如果还没有七牛云账号，需要先申请，通过[邀请链接](https://portal.qiniu.com/signup?code=1hdcuqdt28nyq)注册的同时，我也可以获得额外的5GB流量。

注册认证完成之后，登录七牛云官网，开通对象存储，然后新建空间即可。

> 在新建空间的时候，如果没有特别私密图片的话，访问控制建议选择**公开**，因为选择私有的情况下，外部访问图片相对来说会麻烦点。

空间创建完成之后，系统会随机分配一个测试域名，就是通过这个域名来访问资源。注意，此域名有效期只有一个月。所以没有域名的小伙伴，建议注册一个域名，并且及时备案。然后将域名改为自己的域名。

## 安装七牛云插件

1、进入到博客目录下，执行下面命令，等待安装完成即可。

```shell
$ npm install hexo-qiniu-sync --save
```

2、配置七牛云

在博客根目录下的 _config.yml 文件中添加如下配置：

```yaml
# 七牛
qiniu:
  offline: false
  sync: true
  bucket: bucket_name
  # secret_file: sec/qn.json or C:
  access_key: AccessKey
  secret_key: SecretKey
  dirPrefix: static
  urlPrefix: http://bucket_name.qiniudn.com/static
  up_host: http://upload.qiniu.com
  local_dir: static
  update_exist: true
  image: 
    folder: images
    extend: 
  js:
    folder: js
  css:
    folder: css
```

其中`access_key`和`secret_key`两个密钥在个人中心，密钥管理中。其它配置参数，可以参考[官方文档](https://github.com/gyk001/hexo-qiniu-sync)。

## 使用插件

### 同步资源

配置完成之后，启动本地服务器：

```shell
$ hexo s
```

启动服务之后，会自动生成配置中`local_dir`指定的的文件夹。然后把你需要上传的静态资源存放里面，再使用同步命令：

```shell
$ hexo qiniu sync	# 同步静态资源到七牛空间。可以简写成：hexo qiniu s
```

或

```shell
$ hexo qiniu sync2 # 同步静态资源到七牛空间，且会同步上传那些本地与七牛空间有差异的文件。可以简写成：hexo qiniu s2
```

命令会扫描`local_dir`目录下的文件，同步至七牛空间。此时，就可以在七牛云的个人空间中看到这些静态资源了。

### 如何使用

在我们用`markdown`编写博客的时候，使用下面语法即可使用：

```markdown
{% qnimg test.png %}
```

其中`test.png`就是你上传的静态资源。

更完整的使用方法请见[官方文档](https://github.com/gyk001/hexo-qiniu-sync)。