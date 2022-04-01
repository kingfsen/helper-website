---
title: "本地部署kubernetes.io官方网站"
date: 2020-03-19T16:05:32+08:00
description: "详细介绍如何将kubernetes.io搬迁到本地部署"
thumbnail: "img/kubernetes.png"
Tags:
- kubernetes
- hugo
---

在公司办公网访问**kubernetes.io**官方网站非常慢，为方便工作与学习，可以将其在本地进行部署，快速阅读。**kubernetes.io**项目git地址: https://github.com/kubernetes/website 。
许多开源技术文档均采用Markdown编写，docker、kubernetes也不例外，之后再采用网站生成器Hugo生成静态html进行部署即可。当你克隆该项目时，会发现大小居然接近600M左右，下载还需费点劲。
项目下的makefile文件非常精简，就是使用hugo启动httpServer生成静态站点。

这里不推荐使用官方提供的直接使用hugo内置的httpServer作为web服务，如果按照官方这种形式直接部署，你会发现有一些链接或者功能无法正常使用，尤其是kubernetes.io搜索功能，
它采用的是微软的bingcustomsearch，而我们的服务又在内网，我们只能让bingcustomsearch去官方kubernetes.io网站搜索，然后把搜索结果返回到浏览器，此时返回的结果链接均是
`https://kubernetes.io`，如此一想，当我们要再次点击这些结果时，我们必须在本地配置了host，直接跳转到本地服务，因为需要给本地启动一个https服务，而nginx最好不过了。

# 部署步骤

以Linux为例，首先下载hugo(官网https://gohugo.io/) , 软件包下载地址: https://github.com/gohugoio/hugo/releases ，一定要选择extended版本，我们就选择最新版本下载。

### 安装hugo

```bash
# wget https://github.com/gohugoio/hugo/releases/download/v0.67.1/hugo_extended_0.67.1_Linux-64bit.tar.gz
# tar -xvf hugo_extended_0.67.1_Linux-64bit.tar.gz
# mv hugo /usr/local/bin
# hugo version
Hugo Static Site Generator v0.67.1-4F44227B/extended linux/amd64 BuildDate: 2020-03-15T19:39:16Z
You have mail in /var/spool/mail/root
```

最新版本的hugo对一些系统库有要求，如果系统库版本过低，在执行hugo version命令的时候则会提示错误，其要求的libstdc++.so至少要6.0.21，而当前机器的Linux上最高是6.0.19。
如果本机没有6.0.21版本的libstdc则可以去其他机器拷贝或者网络自行下载一个。

```bash
# ls -l /lib64/libstdc++.so*
# strings /usr/lib64/libstdc++.so.6 | grep GLIBC
```

然后修改libstdc++.so.6软链，指向6.0.21。

```bash
# strings libstdc++.so.6.0.21 | grep GLIBC
# rm libstdc++.so.6 
# ln -s libstdc++.so.6.0.21 libstdc++.so.6
# chmod 744 libstdc++.so.6.0.21
# hugo version
```

### 生成html

进入项目根目录website, 如果你不想管https问题，直接执行 `nohup hugo server --buildFuture &`，则启动了hugo内置的http服务，服务端口默认1313。这种方式不会直接输出生成
的html页面，文档更新也不方便，需要重新启动进程，不建议这种方式。这里不启动http服务，只使用hugo的渲染生成html静态页面。

```bash
# hugo --buildFuture
```

命令执行成功之后，则在当前目录下生成了public目录，里面的内容则可以直接发布到nginx了，在hugo生成静态页面过程中可能会发生一些错误，部分错误可以忽略，部分严重错误可能
会导致hugo渲染失败。由于当前拉取的代码是master分支，其代码本身可能存在一些问题。

- hugo无法读取git信息

	直接找到根目录下的config.toml，该文件是hugo的全局配置文件，将其中的enableGitInfo设置为false即可。
	
- kubernetes-cmd页面404
	
	只需修改`content/en/docs/reference/kubectl/kubectl-cmds.md`文件内容，将其中的链接修改为`/docs/reference/generated/kubectl/kubectl-commands.html` ,
	链接末尾不要打/，因为hugo会把它当成目录处理，然后又找不到index.html，最终报404错误。

### 创建自签名证书

使用openssl为kubernetes.io创建自签名证书，该域名固定，不能修改为其他域名。

```bash
# openssl genrsa -out ca.key 2048
# openssl req -x509 -new -nodes -key ca.key -subj "/CN=kubernetes.io" -days 10000 -out ca.crt 
# openssl genrsa -out server.key 2048
# openssl req -new -key server.key -out server.csr 
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:beijing
Locality Name (eg, city) [Default City]:beijing
Organization Name (eg, company) [Default Company Ltd]:ks
Organizational Unit Name (eg, section) []:ks
Common Name (eg, your name or your server's hostname) []:kubernetes.io
Email Address []:sunjinfu@163.com
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
 
 
# openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 10000
```

### 启动nginx

使用https服务主要是为了解决搜索那个功能，一个静态本地的网站，http直接访问即可，直接将nginx的http以及https服务均开启。

推荐将 config.toml中的baseURL修改为http://kubernetes.io , 然后重新执行 `hugo --buildFuture` 命令生成静态html，再把public目录发布到nginx即可，
同时把上面生成的server.key、server.crt配置到ssl。

```nginx
server {
        listen       80 default_server;
        server_name  _;
        root         /data/website/public;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
	
	server {
        listen       443 ssl http2 default_server;
        server_name  _;
        rewrite ^ http://$http_host$request_uri? permanent;
        ssl_certificate "/data/ssl/server.crt";
        ssl_certificate_key "/data/ssl/server.key";
	}
	
```

```bash
# nginx -t 
# nginx -s reload
```

### 定期同步

```bash
# cd /data/website/ && git pull
# hugo --buildFuture
# nginx -s reload
```

将该脚本添加到Linux的定时任务表中。

然后在本地配上host，浏览器直接访问http://kubernets.io 即可。