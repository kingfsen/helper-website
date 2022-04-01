---
title: "开源评论系统isso"
date: 2019-03-27T16:18:10+08:00
thumbnail: "img/isso.png"
description: "一款超级好用的静态博客评论系统isso"
keywords: "isso"
Tags:
- 评论系统
- 博客插件
Categories:
- 博客组件
---

评论功能是静态博客系统的一个特色，是阅读者交流学习的一种手段。当前国内外都开源出很多评论系统，
包括企业或者个人贡献的，但是很多评论系统临时开放出来，过一段时间则关闭停止服务，所以给自己的博客选择一款稳定
的评论系统非常重要。

对于静态博客来说，由于它缺失后台服务，所以用户的评论数据都是落地在第三方评论系统中，通过第三方提供的JS脚本来实现阅读评论功能。

- 优点<br/>
	博客不用搭建后台服务、数据库等就实现了一个完整的评论系统
- 缺点<br/>
	数据在第三方，如果第三方评论系统停止服务，那么评论数据面临丢失，博客评论功能瘫痪，需要重新更换其他评论组件

总结当前一些流行的评论组件现状

- Disqus<br/>
	Disqus是国外最流行的大厂评论系统，大多静态网站生成器Hexo、Hugo、Jekyll等都默认支持Disqus，但是国内无法访问，必须翻墙。
- 多说<br/>
	国内最流行的评论系统，2017年已停止提供服务。
- 畅言<br/>
	需要备案，严格审核，有限制。
- gitment<br/>
	gitment是由个人提供出来的评论系统，它依托github的仓库issue功能，代理了github的授权，每个页面都必须手动初始化评论，<br/>
	现在这个作者居然不维护了，它的授权代理服务都宕机了。本来感觉每个页面都需要手动初始化评论这个还能忍受，毕竟github的<br/>
	人气非常庞大，
但是个人提供的服务挂掉很久都没人维护了。
- others<br/>
	[talkatv](https://github.com/talkatv/talkatv) 、[juvia](https://github.com/phusion/juvia) 等

总结了一下，将评论数据落地在第三方系统中，还不如自己再搭建一个，数据自己管理和备份，这种评论系统也不占用多少资源，并且有很多现成的开源，
一键部署就可以提供服务了。看了一些评论系统之后，最后我选择了[isso](https://posativ.org/isso/docs/) 。

![en](/blog/isso/003.png)

这里我直接选择了汉化版的isso，其实无非是修改了开源isso的html样式。

![en](/blog/isso/002.png)

- isso有非常丰富的wiki文档
- 后台服务端部署超级简单，提供了API接口
- HTML嵌入JS超级简洁
- isso通过参数配置提供了很多控制功能，比如邮件提醒、评论修改、评论速度等
- 支持匿名评论
- 最重要的一点是isso自带管理后台


isso是用Python语言编写的软件，后端数据库是SQLite，支持Disqus以及Wordpress的数据导入。下面介绍一下如何部署，根据[官网文档](https://posativ.org/isso/docs/) 介绍，花几分钟就能把服务部署起来。
这里推荐大家使用Docker容器方式启动isso服务，这样就避免了环境的配置以及依赖组件的安装。isso的镜像已经提供在docker hub上，地址 https://hub.docker.com/r/wonderfall/isso 。
这个isso镜像ui是汉化版的，是其他开发者上传的，使用任何docker镜像之前，都应该先阅读一遍其Dockerfile文件。

```Dockerfile
FROM alpine:3.9

ARG ISSO_VER=0.12.2

ENV GID=1000 UID=1000

RUN apk -U upgrade \
 && apk add -t build-dependencies \
    python3-dev \
    libffi-dev \
    build-base \
 && apk add \
    python3 \
    sqlite \
    openssl \
    ca-certificates \
    su-exec \
    tini \
 && pip3 install --no-cache "isso==${ISSO_VER}" \
 && apk del build-dependencies \
 && rm -rf /tmp/* /var/cache/apk/*

COPY run.sh /usr/local/bin/run.sh

RUN chmod +x /usr/local/bin/run.sh

EXPOSE 8080

VOLUME /db /config

LABEL maintainer="Wonderfall <wonderfall@targaryen.house>"

CMD ["run.sh"]
```

接着编写isso的启动配置文件isso.conf，大部分配置参数是可选的，这里提供了一份完整的默认配置以及注释`https://github.com/posativ/isso/blob/master/share/isso.conf` 。
启动时必须的最少配置参数

```
[general]
dbpath = /var/lib/isso/comments.db
host = https://example.tld/
[server]
listen = http://localhost:1234/
```

其中host参数是比较重要的，这个host可以配置多个，类似一个白名单的作用，listen是服务器启动的监听地址。isso后端支持配置多个网站同时使用其作为评论系统，
这里提供稍微丰富一点的真实配置。

```text
[general]

dbpath = /db/comments.db
host = https://www.kingfsen.cn/
max-age = 15m
notify = stdout
reply-notifications = false
gravatar = false
gravarar-url = https://www.gravatar.com/avatar/{}?d=identicon

[moderation]

enabled = false
purge-after = 30d

[server]

listen = http://0.0.0.0:8080
public-endpoint = 
reload = off
profile = off

[guard]

enabled = true
ratelimit = 10
direct-reply = 3
reply-to-self = false
require-author = false
require-email = false
```
然后将isso启动配置文件放在机器的/data/isso/config目录下。
拉取镜像

```
docker pull wonderfall/isso
```
启动isso容器

```
docker run -d --name isso -p 8080:8080 -v /data/isso/config:/config -v /data/isso/db:/db wonderfall/isso:latest
```
发送请求验证一下

```
curl http://localhost:8080/count 
```

容器启动之后，isso容器服务端口映射到了宿主机的8080，同时/data/isso/db目录将备份了评论数据。由于我的博客也是部署在当前机器上的nginx中，
接着增加一段配置，nginx代理了isso的服务接口，HTML页面通过域名跨域请求isso服务。

```Bash
location /isso {
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Script-Name /isso;
	proxy_set_header Host $host;
	proxy_set_header X-Forwarded-Proto $scheme;
	proxy_pass http://localhost:8080;
}        
```
因为我们的博客一般默认是location / ，所以这里isso的服务接口都默认以/isso开头，发送请求验证一下，我的nginx配置的域名https://www.youendless.com 。

```
curl https://www.youendless.com/isso/count
```

如果需要开启管理后台需要在isso.conf中增加admin相关配置，密码请记得复杂一点，毕竟通过开发者工具即可获取isso的管理后台。

```
[admin]

enabled = true
password = goodnight;
```

然后浏览器直接访问`https://www.youendless.com/isso/admin`，密码输入正确之后回车即可进入，虽然UI是丑了一点，但是这么精简的东西，实用才是最好的。

![en](/blog/isso/001.png)

isso后端服务没问题，则可以开始在HTML中添加isso的js脚本了。hugo生成的website就是方便，直接新建一个模板isso.html，
通过定义block，在要添加评论的地方直接引入接口，没有模板，则直接把下面的内容添加到需要评论的HTML中即可，有模板，直接将如下内容添加模板中，再把模板嵌入到需要的其他模板中即可。

```JS
<script data-isso="https://www.youendless.com/isso/"
        src="https://www.youendless.com/isso/js/embed.min.js"></script>

<section id="isso-thread"></section>
```
注意data-isso属性，请根据在nginx中转发路径进行配置，如果isso服务与网站在同一个ngxin下，可以直接data-isso="/isso/"，src属性也一样去除域名即可。
如果配置错误，按F12打开浏览器调试面板，可以看见js文件请求失败。
在本地测试的时候，URL是localhost，不是isso中配置的host，所以本地测试会报错。每篇文章下面的评论，就是通过isso实现的，支持匿名评论，随心所欲的交流。


