---
title: "开源Harbor功能结构"
date: 2019-05-20T14:18:25+08:00
description: "详细介绍开源企业级项目harbor的功能结构"
thumbnail: "img/harbor.png"
Tags:
- harbor
- docker
Categories:
- docker
---

在了解Harbor之前，有必要先了解一下Docker开源的Docker Registry，Registry就是用来存储管理Docker镜像的服务，
客户端可以向Registry发起镜像的上传、下拉等请求操作。Docker Registry服务早期是由Docker官方下的docker/registry项目来构建的，
这个项目已经废弃了，取而代之的是Docker官方的另外一个项目docker/distribution，distribution相当于是registry项目的重大变革，
是V2版本，现在Docker架构中的Docker Registry服务指的就是由distribution项目搭建的服务，项目地址: https://github.com/docker/distribution 。

Registry服务纯粹用来管理存储Docker镜像，它没有任何租户的概念，如果仅将Registry用于公司内部构建私有镜像仓库，上层再部署一个OAUTH用户认证系统即可。
Harbor作为一个企业级项目，整合了Docker Registry，具备多租户的支持，更直白的说用户A不能操作或者查看用户B的私有镜像，Harbor中的权限控制是基于项目的，
同一个项目下的所有仓库具备相同的控制权限。Docker Registry是Harbor结构中最核心的一个底层组件，与此同时Harbor以可插拔方式集成了Image Scan、Helm Chart等。

Harbor早期版本(v1.6之前)在github上的vmware组织目录下，从v1.6版本开始成立了单独的一个项目[goharbor](https://github.com/goharbor/harbor)进行开发维护。
在成立单独项目之前，harbor的代码有些地方确实很乱，代码规范以及质量方面有些也不尽人意。从v1.6版本开始，harbor项目进行了代码重构，很多方面都发生了变化，
官方版本更新很快，现在都已经产生v1.8版本了。早期从harbor的v1.4版本为基础构建了企业服务，如今重构之后，我们的企业服务很难跟着升级，
因为早期版本采用的数据库为MySQL，重构之后只支持PostgreSQL，可能考虑到依赖的镜像扫描组件Clair只支持PostgreSQL，而在国内大部分互联网公司的后端数据库都为
MySQL。Harbor在权限控制模块引入了[casbin](https://casbin.org/)，这个casbin支持ACL, RBAC, ABAC等进入控制模式，融入了多种编程语言的支持。

Harbor服务默认所有的组件都是以容器实例提供服务，包括数据库，这种模式一般只用于开发者学习研究，不能作为企业产品进行部署，因为单点故障，数据安全等问题存在，当然
它也有HA部署模式，它采用的是Keepalived，Keepalived主要采用的是虚拟IP漂移技术，经常与Nginx一起搭建主备、主主模式应用服务，但是对于一些中小型企业，它们的技术实力
有限，更好的选择是购买云服务厂商提供的SLB产品。Harbor依赖的软件环境docker、golang、python、docker-compose、git、make等，单台机器硬件至少2核4G以上。

部署单机Harbor服务

- 下载离线包<br/>
	访问https://github.com/goharbor/harbor/releases, 下载对应的harbor离线安装包版本。
- 生成服务配置文件<br/>

	```bash
	tar -zxvf harbor-offline-installer-v1.7.0.tgz
	cd harbor
	./prepare --with-clair
	```
	
	prepare执行之后，会从harbor/common/templates下选择对应的模板，利用harbor.cfg中的配置进行渲染，然后各组件启动的配置文件生成到harbor/common/config目录中。
	--with开始的参数都是harbor中对应的可选的插拔式服务，`--with-clair`安装镜像扫描，`--with-chartmuseum`安装Chart仓库，用于支持helm charts存储。
- 启动服务<br/>
	
	```bash
	./install.sh --with-clair
	```
	
	服务启动成功之后，各容器的日志均被收集在/var/log/harbor目录下面，服务正常之后具备健康检查的容器全部处于healthy状态。
	
	![harbor](/blog/harbor_service_deploy/001.png)
	
- 停止服务<br/>
	
	```bash
	docker-compose -f docker-compose.yml
	```
	
	如果启动的时候同时开启了镜像扫描服务，此时也许停止容器，执行如下
	
	```bash
	docker-compose -f docker-compose.yml -f docker-compose.clair.yml down
	```
	
	如果修改了配置文件，需要重启服务，可以直接再次执行`./install.sh --with-clair`命令即可。
	
如果直接从github上拉取了源码，可以在源码的基础上进行打包，但是执行打包的机器必需具备访问外网的能力，因为它需要去官方或者第三方下载一些依赖包或者源码，进入harbor项目的根目录执行打包命令即可。

1.6版本之前的打包命令

```bash
make package_offline GOBUILDIMAGE=golang:1.7.3 COMPILETAG=compile_golangimage \
CLARITYIMAGE=vmware/harbor-clarity-ui-builder:1.2.7
```

如果需要安装镜像扫描等可选组件，则需加上参数CLAIRFLAG=true，这样打包的时候会去github拉取Clair的源码，构建Clair镜像。

```bash
make package_offline GOBUILDIMAGE=golang:1.7.3 COMPILETAG=compile_golangimage \
CLARITYIMAGE=vmware/harbor-clarity-ui-builder:1.2.7 CLAIRFLAG=true
```

官方的这种打包流程太过繁琐耗时，构建一次至少需要20分钟，并且由于国内网络原因，很容易依赖包下载失败，所以作为企业级产品的时候，
我们首先就是优化这个打包构建镜像的流程，基于我们自己的github仓库，打包上传30秒之内完成，harbor-clarity-ui-builder是官方做好
的一个用于构建Harbor UI页面的镜像，每个版本可能不一样，请查阅官方文档。


Harbor作为企业级项目进行部署时，重要的存储中间件，肯定都要抽离出来。云数据库、对象存储、负载均衡均可直接从云服务厂商购买，下面是一个简单的架构模型。

![deploy](/blog/harbor_service_deploy/002.jpg)

上图只是一个简单的模型，至于是否分内外网LB无关紧要，看具体的服务流量，重要的是把存储层都抽取出来了。

- **MySQL** Harbor的主体数据库(v1.6之前)
- **PostgreSQL** 镜像扫描组件Clair的数据库，主要用于存储CVE数据源以及镜像层扫描记录
- **S3/KS3/OSS** 对象存储中间件用于存储Docker镜像文件或者Helm Chart包文件
- **redis** 用于registry缓存，多实例部署Harbor时用户session储存同步，当然1.6版本之前可以为空，关闭registry缓存，修改源码利用mysql存储用户session


下面看一下在Harbor官方基础上稍作修改的功能结构图，主要就是提取出存储层。这里仍然以v1.5版本为例，
因为后面的版本逐渐集成了很多组件，但是老版本的核心组件大部分仍然在新版本中延续，实线箭头代表的是请求方向。

![harbor_overview](/blog/harbor_service_deploy/003.jpg)

橙色虚线框是Harbor集成的镜像扫描功能，绿色虚线框代表的是Harbor核心基础服务，主要包含Proxy、Adminserver、Jobservice、UI、Registry、Log六个组件，
这六个容器必须成功启动才能构成一个完整的Harbor核心基础服务。

- **Proxy** 容器内部启动了一个Nginx服务，它反向代理了所有进入Harbor的请求，然后根据URL转发到Harbor内部其它各相关的组件。
- **UI** Harbor内部业务核心组件，它又主要包含了web ui、token、api、webhook等模块。web ui是harbor的用户界面，api是harbor对外提供的http rest服务，
	webhook当前仅用于处理两种通知，Clair已扫描的镜像层包含的CVE被删除或者更新以及Registry镜像上传下载等操作，token基于JWT生成访问Registry服务的bearer，
	bearer token中包含了资源对象的权限控制。
- **Registry** 基于docker官方distribution项目构建的镜像存储管理服务，所有进入Registry的请求都必须携带一个有效的认证token，典型的例子就是Docker Client的docker login、docker pull等相关请求。
- **Adminserver** Harbor系统配置管理端，操作数据库中的配置表(properties)。 
- **Jobservice** Harbor中镜像扫描、远程仓库复制都是以job的形式启动一个服务，每个job都有不同的状态，以状态机模式进行处理，当前只有这2种job。
- **Log** Harbor内部容器日志收集组件，内部运行的是rsyslog，docker日志引擎默认支持rsyslog，日志文件通过docker-compose文件配置，默认映射在宿主机的/var/log/harbor目录下。
