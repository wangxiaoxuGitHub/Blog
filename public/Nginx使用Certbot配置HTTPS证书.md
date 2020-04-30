---
title: Nginx使用Certbot配置HTTPS证书
date: 2020-04-25 15:40:24
tags:
- Nginx
- Https
- Certbot
categories: Linux
top 10
---
![20200426131416](https://img.yjll.art//img/20200426131416.jpg)

随着互联网的发展，数据安全问题越来越收到重视，Chrome 将没有配置 SSL 加密的 HTTP 网站标记为不安全，HTTPS是未来互联网发展的趋势，但是想要网站支持HTTPS网站必须有Certificate Authority颁发的证书，一般CA签发的证书都是收费的。本文将介绍如何使用免费的授权证书并自动续签。

Let’s Encrypt 是 一个叫 ISRG （ Internet Security Research Group ，互联网安全研究小组）的组织推出的免费安全证书计划。ISRG 的发起者 EFF （电子前哨基金会）为 Let’s Encrypt 项目发布了一个官方的客户端 Certbot ，利用它可以完全自动化的获取、部署和更新安全证书。

<!-- more -->

### 准备


我们已经安装了Nginx，并且使用默认的80端口，例如我的Hexo博客配置，Certbot会更改nginx的配置文件，操作前最好备份一下nginx.conf。
``` 
    server {
        listen 80;
        server_name blog.yjll.art;
        location / {
            include /etc/nginx/mime.types;
            root /var/www/blog/public;
            index index.html;
        }
    }

```

我们先安装Certbot,我是用的是CentOS7，就直接使用yum安装啦。

```
yum install python2-certbot-nginx
```

### 通过Certbot获取配置证书

安装完成后，输入命令

    certbot --nginx

期间会要求输入邮箱信息，确认协议和域名等，根据提示输入就好。我们看一下生成好的nginx.conf，原来的80变成了443端口，增加了证书配置。

```
    server {
        server_name blog.yjll.art;
        location / {
            include /etc/nginx/mime.types;
            root /var/www/blog/public;
            index index.html;
        }

        listen 443 ssl; # managed by Certbot
        ssl_certificate /etc/letsencrypt/live/yjll.art/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/yjll.art/privkey.pem; # managed by Certbot
        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
    }

```

为了原来的http连接也可以正常使用，我们可以通过Nginx将http请求重定向到现在的端口上

```
    server {
    listen       80;
    rewrite ^(.*) https://$host$1 permanent;
    }
```

好了，可以在浏览器访问HTTP和HTTPS链接试一试了。



### 自动续期

使用起来确实挺方便的，但是Let’s Encrypt发布的证书期限只有三个月，在证书到期前我们要重新申请证书，`certbot`也提供了续订的功能

```
certbot renew
```

也就是说在证书过期前要运行一下上边的命令，并且重新加载nginx的配置文件，这部分我们可以配置在定时任务中。

```
45 4 1 * * certbot renew --force-renew --renew-hook "nginx -s reload"
```

每一个月第一天，重新申请证书


### 参考资料

https://certbot.eff.org/

https://learnku.com/articles/19999
