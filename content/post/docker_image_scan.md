---
title: "Docker镜像扫描原理"
date: 2019-05-05T15:18:22+08:00
description: "详细分析Docker镜像扫描处理过程与原理"
thumbnail: "img/image_scan.png"
Tags:
- Docker
- 镜像扫描
Categories:
- docker
---

初次听说镜像扫描想必很多疑惑，因为不明白其中的处理过程与原理。以Windows为例，它的控制面板以及一些第三方安全软件(三六零等)总能展示当前系统已安装的软件包，
当然绿色解压版则无法探测，它们无非是读取了操作系统的一些系统资源文件。Linux系统也不例外，通过软件管理包rpm、dpkg等安装的组件都会记录在系统相关文件中，
那么镜像扫描的原理就不难明白了，首先获取系统已安装组件清单文件，然后解析清单文件，将清单中的组件名、版本等关键元素与CVE数据中心进行比对，最后输出比对结果。


![docker_image_scan](/blog/harbor_image_scan/002.png)


镜像扫描作为一个插拔式的扩展功能，更多的是集成在Docker镜像仓库服务端(Docker Hub)，而不是Docker Client。当用户将镜像上传到镜像仓库时，系统自动或者用户手动
触发镜像扫描功能，将当前镜像的风险漏洞展示出来，由用户自己决定是否下载或者系统直接隐藏包含很多高风险等级漏洞的镜像。

当前有开源实现方案，对于企业级方案，最常用的是使用harbor来搭建镜像仓库服务，使用Clair来支撑Docker镜像扫描功能，下面这个功能结构图是在开源harbor基础上做了一些
拆分改造，然后作为一款企业级产品，对外面向用户。注意，图中的harbor结构图是1.5之前版本，1.6之后的版本其结构已经发生很大变化。

![harbor_clair](/blog/harbor_service_deploy/003.jpg)


在了解镜像扫描之前，这里先简单说下镜像的概念，镜像就是由许多Layer层组成的文件系统，重要的是每个镜像有一个manifest，
这个东西跟springboot中的manifest是同一个概念，就是文件清单的意思。一个镜像是由许多Layer组成，总需要这个manifest文件来记录下到底由哪些Layer层联合组成的。
要扫描分析一个镜像，首先你就必须获取到这个镜像的manifest文件，通过manifest文件获取到镜像所有Layer层的地址digest，
digest在docker镜像存储系统中代表的是一个地址，类似操作系统中的一个内存地址概念，通过这个地址，可以找到当前Layer层文件的内容，
这种通过digest可寻址的设计是hub v2版本的重大改变。在docker hub储存系统中，所有文件都是有地址的，
这个digest就是由某种高效的sha算法通过对文件内容计算出来的，类似在linux上的md5sum命令，执行`md5sum xx.txt`，即可得到一个摘要。

[Clair](https://github.com/coreos/clair)是CoreOS旗下的一个项目，用golang语言编写实现，提供了http rest接口。Clair接收镜像扫描请求时，并不会要求客户端传递镜像
文件，而是让客户端告诉Clair当前这个镜像层所在URL，这个URL一般是镜像层在镜像仓库中的下载链接，当然请求镜像仓库时所需的Token等认证参数，客户端必须全部传递给
Clair，Clair会带着Token等认证参数直接请求URL链接下载镜像层。如果一个镜像有5层，那么客户端必须向Clair请求5次，也就是说Clair的扫描处理是基于镜像层来进行处理的。

Clair的基本功能结构图如下：

![docker_image_scan](/blog/harbor_image_scan/003.png)


- **API** Clair对外提供的http rest api
- **detect** Clair探测器，探测镜像操作系统、已安装的组件版本等
- **datastore** 操作数据库，类似Java中的dao层
- **postgres** Clair当前版本唯一支持的数据库PostgreSQL
- **notifier** 当某个具体的CVE漏洞有更新或者删除时，通知webhook
- **webhook** 在Clair中注册的通知URL
- **updater** Clair中负责更新CVE数据源的定时调度
- **fetch** 从各开源社区拉取CVE数据源，插拔式、可自定义扩展

红色虚线框是Clair的CVE漏洞数据源同步功能，数据源同步的详细介绍可以参考另外一篇文章[Docker镜像扫描之CVE漏洞数据源](/post/cve/)。

首先看下Clair镜像扫描请求中的几个关键参数：

```go
type Layer struct {
	Name             string                      
	Path             string            
	Headers          map[string]string 
	ParentName       string            
	Format           string            
}
```


- **Name** 在Clair中是全局唯一的，每个镜像的每一层在Clair中都是用唯一标识符Name进行识别的，不同的层必须保证生成不通的标识，必填
- **Path** 镜像层在镜像仓库中的完整URL下载链接，必填
- **Headers** Clair请求Path下载镜像层文件内容时必须携带的认证参数
- **ParentName** 镜像层的父类镜像层Name，从镜像的最底层往上的每一层都有父镜像层，为了获取镜像的完整扫描结果，请务必传递父镜像层
- **Format** 对于Docker镜像，Format的值为Docker(不区分大小写)，Format很重要，在Clair中通过Format参数选择指定的文件提取器来提取文件，当前Clair文件提取器支持
docker、aci两种，必填

下面详细介绍Clair处理镜像扫描请求的详细流程:

- 检查当前层是否存在

	根据Name检查当前层是否已在DB中，已存在并且版本无更新则无需处理，如果不存在，则构造db.Layer对象，此时如果传递了ParentName，则校验ParentName所对应的层是否存在。

- 读取镜像层文件内容
	
	Clair支持从磁盘读取镜像层文件或者通过http/https协议发送GET请求下载镜像层文件内容，镜像层的内容在镜像仓库实际就是一个Blob文件，
	下面就是digest为087a57faf9491b1b82a83e26bc8cc90c90c30e4a4d858b57ddd5b4c2c90095f6的镜像层在镜像仓库的存储路径与方式。
	
	```bash
	[root@vm1 087a57faf9491b1b82a83e26bc8cc90c90c30e4a4d858b57ddd5b4c2c90095f6]# pwd
	/data/registry/docker/registry/v2/blobs/sha256/08/087a57faf9491b1b82a83e26bc8cc90c90c30e4a4d858b57ddd5b4c2c90095f6
	[root@vm 087a57faf9491b1b82a83e26bc8cc90c90c30e4a4d858b57ddd5b4c2c90095f6]# ls -lrt
	total 4240
	-rw-r--r-- 1 10000 10000 4340083 Mar  7 14:44 data
	[root@vm 087a57faf9491b1b82a83e26bc8cc90c90c30e4a4d858b57ddd5b4c2c90095f6]# 
	```

- 提取信息文件

	这一步是非常重要的，也是至关重要的，Clair根据Format参数采用指定的文件提取器，Docker镜像自然采用Docker结构文件提取器。Clair需要提取的文件主要有2种，
	分别是操作系统版本信息文件以及安装包清单文件，此时Clair并不知道当前镜像系统是基于哪个Linux发行版，它只能按照顺序逐个探测，所以有必要先了解下各系统
	版本文件路径。
	
	| Linux发行版  | 系统版本文件 |
	| ------------- | ------------- |
	| CentOS/Red-Hat  | /etc/oracle-release, /etc/centos-release, <br/>/etc/redhat-release, /etc/system-release  |
	| Debian/Ubuntu  | /etc/os-release, /usr/lib/os-release  |
	| Ubuntu Precise  | /etc/lsb-release  |
	| Debian unstable  | /etc/apt/sources.list  |
	| Alpine  | /etc/alpine-release  |
	
	
	安装包清单文件
	
	| 软件包管理器  | 软件包清单文件 |
	| ------------- | ------------- |
	| apk  | /lib/apk/db/installed  |
	| dpkg  | /var/lib/dpkg/status  |
	| rpm  | /var/lib/rpm/Packages  |
	
	明确了以上待提取文件列表之后，Clair则开始解压镜像层归档文件，遍历归档文件列表，通过文件名称进行识别，提取指定文件内容，最终得到一个类型为map[string][]byte
	的filesMap，key为文件路径，value为文件对应的字节内容。

- 探测Namespace

	Clair中的Namespace概念即操作系统版本，如centos:7.1，循环使用各Linux发行版的探测器去探测上一步提取出的filesMap，有一个探测器提取出Namespace则表示成功。
	以我的一台CentOS服务器为例：
	
	```bash
	[root@vm ~]# cat /etc/centos-release
	CentOS Linux release 7.3.1611 (Core) 
	```
	
	CentOS探测器使用正则匹配文件内容，最终提取出Namespace为centos:7，因为软件包产生的CVE漏洞都是基于Namespace产生的。
	
- 探测安装包
	
	软件包管理器有apk、dpkg、rpm三种，每种包管理器的清单文件路径不一致。对于CentOS，RedHat，软件包安装清单文件路径/var/lib/rpm/Packages，Clair通过执行rpm相关命令
	查看当前已安装的全部软件包版本信息。
	
	```bash
	[root@vm rpm]#rpm -qa --qf "%{NAME} %{EPOCH}:%{VERSION}-%{RELEASE}\n"
	pciutils-libs (none):3.5.1-1.el7
	python-urwid (none):1.1.1-3.el7
	memcached 0:1.4.15-10.el7
	libogg 2:1.3.0-7.el7
	pexpect (none):2.3-11.el7
	ipmitool (none):1.8.15-7.el7
	boost-thread (none):1.53.0-26.el7
	python-jwcrypto (none):0.2.1-1.el7
	ntp (none):4.2.6p5-25.el7.centos
	libverto-tevent (none):0.2.5-4.el7
	bind-utils 32:9.9.4-37.el7
	rsync (none):3.0.9-17.el7
	gettext-libs (none):0.18.2.1-4.el7
	ldns (none):1.6.16-10.el7
	ppp (none):2.4.5-33.el7
	copy-jdk-configs (none):1.2-1.el7
	...
	```
	rpm命令执行成功之后，Clair则开始逐行解析命令输出内容，提取features，如pciutils-libs，版本则为3.5.1-1.el7。
	
	对于debian、ubuntu系统，安装包清单文件路径/var/lib/dpkg/status，该文件为一个纯文本文件，直接读取文件内容，逐行解析即可，下面只列举了2个安装包记录展示。
	
	```bash
	Package: sed
	Essential: yes
	Status: install ok installed
	Priority: required
	Section: utils
	Installed-Size: 799
	Maintainer: Clint Adams <clint@debian.org>
	Architecture: amd64
	Multi-Arch: foreign
	Version: 4.4-1
	Pre-Depends: libc6 (>= 2.14), libselinux1 (>= 1.32)
	Description: GNU stream editor for filtering/transforming text
	 sed reads the specified files or the standard input if no
	 files are specified, makes editing changes according to a
	 list of commands, and writes the results to the standard
	 output.
	Homepage: https://www.gnu.org/software/sed/
		 
	Package: libsmartcols1
	Status: install ok installed
	Priority: required
	Section: libs
	Installed-Size: 257
	Maintainer: Debian util-linux Maintainers <ah-util-linux@debian.org>
	Architecture: amd64
	Multi-Arch: same
	Source: util-linux
	Version: 2.29.2-1+deb9u1
	Depends: libc6 (>= 2.17)
	Description: smart column output alignment library
	 This smart column output alignment library is used by fdisk utilities.
	```
	最终Clair提取`Package`以及`Version`字段，获取libsmartcols1 2.29-1+deb9u1以及sed 4.4-1等安装包属性信息。
	Clair将每一个提取的feature构造为一个database.FeatureVersion对象，同时将上一步探测到的Namespace填充到每一个FeatureVersion对象中。
	
	```go
	f := &database.FeatureVersion{
		Feature: database.Feature{
			Name: sed,
			Namespace: database.Namespace{
				Name: centos:7,
				VersionFormat: rpm,
			},
		},
		Version: 4.4-1
	}
	```
	
- 数据持久化

	将解析的Layer信息全部保存到PostgreSQL数据库中。
	
Post请求Clair镜像扫描(/layers)成功之后，之后则可以通过Name向Clair发起GET(/v1/layers/6d33c67920b31f6dcea328762fe1a814de928a185d9397f61b15a278c17184f2?features)
请求，这个Name可以是最顶层的镜像层唯一标识符，Clair会把该镜像层、父层以及所有的间接父层的CVE信息全部输出，也就是整个镜像的CVE信息。

看完整个Clair的处理流程，发现Clair只是提取了Docker镜像层中的两个文件而已，可以在Clair的基础上扩展，做一个插件安装在任何Linux系统，定时将系统文件或者安装包清单文件
上传给Clair，实时监控扫描系统，而不局限于集成在Docker hub仓库扫描Docker镜像，可以直接在Docker Client上传镜像之前进行扫描处理。