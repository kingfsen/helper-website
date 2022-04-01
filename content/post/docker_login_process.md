---
title: "Docker Login执行流程与原理"
date: 2019-03-16T08:54:16+08:00
thumbnail: "img/docker.jpg"
description: "docker login执行原理与流程"
Tags: 
- harbor
- docker
Categories: 
- docker
---

docker安装的时候已经同时安装了docker client，通过命令docker version即可查看客户端以及服务端的版本信息，通过执行命令`docker version`查看docker版本信息。docker最近暴露的runc漏洞CVE-2019-5736，企业环境请安装18.09.2以上版本。
```Docker
Client:
 Version:           18.06.0-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        0ffa825
 Built:             Wed Jul 18 19:08:18 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.0-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       0ffa825
  Built:            Wed Jul 18 19:10:42 2018
  OS/Arch:          linux/amd64
  Experimental:     false
```

docker login等命令行则是用来对harbor服务发起http请求，早期版本的harbor registry服务url为/v1，后期harbor改为了/v2，所以如果你的docker版本太老，则会请求到/v1，自然会一直报错，查看harbor中Proxy的配置。

```Bash
    location /v1/ {
      return 404;
    }

    location /v2/ {
      proxy_pass http://ui/registryproxy/v2/;
      proxy_set_header Host $$http_host;
      proxy_set_header X-Real-IP $$remote_addr;
      proxy_set_header X-Forwarded-For $$proxy_add_x_forwarded_for;
      
      # When setting up Harbor behind other proxy, such as an Nginx instance, remove the below line if the proxy already has similar settings.
      proxy_set_header X-Forwarded-Proto $$scheme;
      proxy_buffering off;
      proxy_request_buffering off;
    }
```
本篇文章主要讲解docker login、docker pull等命令底层的执行流程以及相关原理，不知道多少人对docker client最基本的login命令有误区，简单的以为这是去登录harbor仓库，可能搞了多年的一些程序员可能也掉进这个误区，不去查阅官方文档，仔细分析，还真以为这个命令只是一个简简单单的用户名密码登录功能。
```Bash
docker login hub.xxx.com
Username: 2000014559
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```
登录成功之后，根据提示信息可以知道，它把用户名密码存放在/root/.docker/config.json中，进入/root/.docker查看config.json

```Json
{
        "auths": {
                "hub.xxx.com": {
                        "auth": "MjAwZZZxNDU1OTpSb299dxedssDU2"
                }
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/18.06.0-ce (linux)"
        }
}
```
在linux执行解码一看究竟

```
echo 'auth中的内容'|base64 --decode
```
解码后即是username:password。下面看看harbor服务端收到的请求，从harbor的nginx服务看到，docker login操作一共发出了三次http的请求。

```HTTP
Sep 14 14:08:09 172.18.0.1 proxy[13966]: 10.69.56.148 - "GET /v2/ HTTP/1.1" 401 87 "-" 
Sep 14 14:08:09 172.18.0.1 proxy[13966]: 10.69.56.148 - "GET /service/token?account=2000014559&client_id=docker&offline_token=true&service=harbor-registry HTTP/1.1" 200 897 "-" 
Sep 14 14:08:09 172.18.0.1 proxy[13966]: 10.69.56.148 - "GET /v2/ HTTP/1.1" 200 2 "-"
```
harbor服务几个关键服务组件为nginx、ui、registry，nginx作为harbor服务的反向代理，接收客户端所有的请求，然后发布到其他组件，而registry服务要求所有进入它的请求都必须携带一个合法的token，否则返回401未授权。

第一个get请求/v2首先通过nginx转发到ui服务，经过ui的handleChain处理之后，又转发到registry服务，registry服务经过验证，发现请求header中未传递token，直接返回401。
docker client收到401之后，马上请求/service/token，这个服务是nginx直接转发给ui组件，ui组件检测到请求类型不是repository操作，则会强制要求用户名密码不能为空，下面请看docker login 与docker pull获取token具体的参数。

- docker login发起的请求

	```HTTP
GET /service/token?account=2000014559&client_id=docker&offline_token=true&service=harbor-registry
	```

- docker pull发起的请求 

	```HTTP
 GET /service/tokenaccount=2000014559&scope=repository%3Alibrary%2Fprome%2Fnodeexporter%3Apull&service=harbor-registry
	```
  
docker login的时候没有scope，会强制鉴权，即username:password然后base64编码之后，塞在请求头中的Authorization中。ui鉴权通过之后，获取private_key，
利用jwt算法生成一个token，这个token还有个有效期。

- jwt完整token示例

	```Text
  eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImtpZCI6IlFDWTI6Qkk3RTpLMkpWOldYR1g6SEZGSDpMVUJOOlo3Q0k6TFlETDpEUkhaOlBBR1M6RFpQTDpUM1JQIn0.eyJpc3MiOi
  JoYXJib3ItdG9rZW4taXNzdWVyIiwic3ViIjoiMjAwMDAxNDU1OSIsImF1ZCI6ImhhcmJvci1yZWdpc3RyeSIsImV4cCI6MTUzNjkwNzA4OSwibmJmIjoxNTM2OTA1Mjg5LCJpYXQi
  OjE1MzY5MDUyODksImp0aSI6IlJuVzhIdGdqdThiQkswbUsiLCJhY2Nlc3MiOm51bGx9.LQveGSQqQ-XvRBYewY2ojqnlNfqhCUhy99s9oQ5WTJ5zdZInBiLJPMKfZReTaZriwTB3c
  CYwUAvaolLcthLdPZlJil48gG5hWRBeNoiJJNcNHY054wrxhhcJq9v0xcfdJnJAK_sVeuz1pA4v99Z-MMMBYLXHp0mx_sqy1kCJiGrtwU4JshVgG4NBAQLHB8atpdjvfhimFxHfs8-
  oRyY4EjMJx5-SKYEQadA4SR53VhYaLlL10LBhMSyZj-1C54P1GCf6zn2HHiqaRFOur8zQbX7nNgVz2WszgSCIU9gmxkzS2jD6QWJdfJKvBCHeY8lmoNxGROJvoYAppK6h_f3edOQjg
  tfrdxyLcneQEtNoVRUzXPwPxtJIh1ISm8xbF0NVfuV2Ntbn4nnUTcJBEw8y4sTyb-l5J8XFzs8idFdN4a7JPSlne4L4lm6pPJsKXTgUp4vFdNvN8lY2pQmtUvEKFPZRgGVFoyIvo8U
  5KoKX120CGMsXiZ89k_bm98mFwbq2S4hI2jRujUTNopN0qG3TqK2dl6cF_YzoGEt9eU7cblPGpHbE5bqxsXojXsyxn3R8ErmhDo3__-2Z9vyKWTTgy8MLVSj-bMsXfeM3oT6fdNoFH
  tYxYwQ9FrAiMOO7cirZAETGN5bwoeNRCF1UCuvJgQpzvzH-PKzuez91OdI8NtE
	```

docker client获取到token之后再去重复第一步的请求，这时候响应结果状态码为200，docker client才认为用户名密码是有效的，它才会提示Login Success，同时保存在/root/.docker/config.json中，用于后面操作镜像仓库的时候，让用户免输入。从上面的分析看，这里所谓的docker login，并不是去调用harbor服务的登录接口生成session，只是用来验证一下是否可以正确获取到token，当然这时候的用户名密码必须真实存在。  

docker pull操作也是一样的，每一次的pull操作都是发送三个请求，与docker login不同的就是在请求生成token时把仓库权限写入了token中，你请求的是docker pull操作，ui服务会判断当前仓库如果是公开的，则具备读R权限，如果用户是当前仓库管理员，则拥有RWM权限，如果是仓库的开发者，则拥有RW权限，然后把当前相关的信息都写入到这token中，当registry服务接收到这个请求的时候，首先利用root.crt解密token中相关的信息，然后对当前操作进行权限校验。

利用postman操作一下，首先获取servicetoken

![get_servicetoken](/blog/docker_login_process/001.jpg)

然后拿到这个token，把这个token放在请求头中的Authorization中，前面不再是Basic，而是Bearer。这里只做一个演示，并不是跟前面一个完整的请求。

![get_servicetoken](/blog/docker_login_process/002.jpg)

了解了这些基本的docker client命令背后的原理，有助于harbor服务端的开发部署。