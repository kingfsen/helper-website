+++
title = "Harbor仓库镜像扫描原理"
date = "2019-03-11 15:59:41"
thumbnail = "img/clair.png"
description = "Harbor仓库镜像扫描原理分析"
Tags = ["harbor", "clair", "镜像扫描"]
Categories = ["docker"]
+++


初次听说镜像扫描的人肯定有很多疑惑，总会想原理是什么呢？我们可以先思考下，windows控制面板、包括一些第三方软件比如三六零等，它们都能获取系统安装的软件以及版本，当然绿色解压版他们就无法识别，获取软件版本之后，他们就能提供一些升级的版本、以及当前软件版本的漏洞列举出来，由此可见，系统安装了哪些版本的软件，第三方都可以通过系统相关文件获取，那么Linux系统也一样。

Harbor仓库中的镜像扫描这个功能，看似很高大上，其实等你了解了它的底层原理与流程，你就会发现就是做了那么一件事而已，用通俗的一句话概括，就是找到每个镜像文件系统中已经安装的软件包与版本，然后跟官方系统公布的信息比对，官方已经给出了在哪个系统版本上哪个软件版本有哪些漏洞，比如Debian 7系统上，nginx 1.12.1有哪些CVE漏洞，通过对逐个安装的软件包比对，就能知道当前这个镜像一共有多少CVE。下面就Harbor的具体流程进行简单介绍，让你对这个功能了如指掌。

Harbor仓库中的镜像扫描功能是扩展功能，如果需要，只需启动的时候带上参数`./prepare --with-clair && ./install.sh --with-clair`。Harbor镜像扫描功能依赖的第三方项目是coreos旗下的clair，项目地址：https://github.com/coreos/clair ,简单介绍下clair系统中几个核心功能结构。

### Clair功能结构

![clair功能结构](/blog/harbor_image_scan/001.jpg)

- API是clair对外提供的http api，harbor请求镜像扫描则是请求clair提供的http接口之一。
- notifier用来调用外部系统的webhook，当软件包的CVE被更新或者删除的时候，调用webhook通知第三方系统，消息中只有一个id。
- updater是clair中的定时任务，定期从各开源社区获取CVE数据，由于各开源社区维护的CEV结构各自不同，clair的每个fetcher只能fetch一个CVE数据源，然后调用对应的parser解析CVE数据，然后统一格式化，最后存入postgresql数据库中。clair系统中很多模块都是可以扩展的，提供了扩展接口，比如数据源、解析器等。系统首次启动去获取CVE数据源，由于全量更新耗时差不多1小时左右，后期都是增量更新，更新频率可以自由设置(单位小时)。
### 扫描触发机制
在了解镜像扫描之前，这里先简单说下镜像的概念，镜像就是由许多Layer层组成的文件系统，重要的是每个镜像有一个manifest，这个东西跟springboot中的manifest是同一个概念，就是文件清单的意思。一个镜像是由许多Layer组成，总需要这个manifest文件来记录下到底由哪些Layer层联合组成的。要扫描分析一个镜像，首先你就必须获取到这个镜像的manifest文件，通过manifest文件获取到镜像所有Layer层的地址digest，digest在docker镜像存储系统中代表的是一个地址，类似操作系统中的一个内存地址概念，通过这个地址，可以找到当前Layer层文件的内容，这种通过digest可寻址的设计是harbor v2版本的重大改变。在docker hub储存系统中，所有文件都是有地址的，这个digest就是由某种高效的sha算法通过对文件内容计算出来的，类似在linux上的md5sum命令，执行md5sum xx.txt，即可得到一个摘要。
![harbor扫描功能](/blog/harbor_image_scan/002.jpg)

上图中，左边是harbor系统，右边是clair系统，左边的harbor只给出了与镜像扫描有交互的模块，并不是完整的功能结构，如需了解harbor的完整功能结构请参考另一篇文章 [部署Harbor镜像仓库服务](/post/deploy_harbor_service) 。在harbor系统中有几个镜像扫描的触发点，当然根据参数设置，可以关闭系统触发扫描。

- 每天定时全量扫描
- docker client执行docker push推送镜像成功之后，registry发送event通知harbor，harbor对当前的推送成功的镜像进行扫描
- 系统启动的时候全量扫描
- http api请求扫描指定的镜像

### Harbor请求流程
下面以用户调用harbor的http api触发镜像扫描为例，详解流程原理，先说harbor这一侧流程，后面再说clair接收到扫描请求的流程。

- 用户请求harbor的/api/repositories/nginx/tags/1.12.1/scan接口，发起对版本为1.12.1的nginx进行镜像扫描。
- UI模块对用户操作权限认证通过之后，发起对Jobservice的post请求。
- Jobservice调用Registry校验该镜像的manifest成功之后，获取该manifest的digest，然后在数据库中生成一个初始状态的扫描job记录，同时把这个job加入到定时调度器中。
- Jobservice用digest为唯一键，在image_scan_overview表中插入一条存放记录，待获取到扫描结果之后更新对应的扫描结果字段，Jobservice将job状态修改为Running，状态机模式开始处理。
- Jobservice调用UI模块生成请求registry的token令牌，该token中包括对该镜像的一个操作权限(push、pull)，同时根据digest调用Registry获取镜像的manifest文件内容，解析manifest。
- 构造请求扫描系统Clair的请求参数，将镜像的每一层参数封装成ClairLayer结构。

	```Go
	l := models.ClairLayer{
				Name:    fmt.Sprintf("%x", sha256.Sum256([]byte(shaChain))),
				Headers: tokenHeader,
				Format:  "Docker",
				Path:    utils.BuildBlobURL(registryEndpoint, iz.Context.Repository, string(d.Digest)),
			}
			if len(iz.Context.layers) > 0 {
				l.ParentName = iz.Context.layers[len(iz.Context.layers)-1].Name
			}
	```
	通过相应的算法sha256.Sum256计算Name，保证镜像的每一层Layer在Clair系统中都具备唯一标志，Headers则包含了clair请求Registry必备的请求头信息，最重要的是上一步生成的token令牌，Path则是镜像的每一层在Registry中的下载地址，除了镜像的基础层，其他每一层都维护着父子关系，参数demo如下:

	```Text
	Name:    sha256:7d99455a045a6c89c0dbee6e1fe659eb83bd3a19e171606bc0fd10eb0e34a7dc
	Headers: tokenHeader,
	Format:  "Docker",
	Path:    http://registry:5000/v2/nginx/blobs/7d99455a045a6c89c0dbee6e1fe659eb83bd3a19e171606bc0fd10eb0e34a7dc
	ParentName: a55bba68cd4925f13c34562c891c8c0b5d446c7e3d65bf06a360e81b993902e1
	```
	这里的Name并不是Layer的digest，每一层的Name都是它的digest与它所有的直接父级以及间接父级digest拼接起来，再通过sha256.Sum256生成的。所有的参数准备完毕之后，循环每一个layer，逐层调用Clair的http接口请求扫描，每个请求都是同步等待结果，只要有一个Layer请求返回失败，则马上退出状态机，同时更新该job的状态为error，所有Layer请求都成功只后，则开始用最后一层的Name请求Clair获取扫描结果，Clair会返回该Name对应的所有parent的扫描结果信息，获取扫描结果之后，Jobservice退出状态机，同时更新job状态为success，扫描结果表image_scan_overview，Harbor的页面则可以直接从自身数据库获取扫描结果了。

### Clair扫描流程

- clair接收harbor发起的http post镜像扫描请求，同时进行基本的参数校验。
- clair利用请求参数中的token headers，直接对Registry的path发起请求，下载镜像层文件，然后解析镜像文件内容，得到一个filesMap。
- 探测镜像操作系统，遍历解压后的文件目录，探测操作系统文件路径。 首先要了解各Linux发行版的一些基础文件，比如系统版本、安装的软件包版本记录等文件，比如centos

	```
	centos：etc/os-release,usr/lib/os-release
	```
	查看文件/etc/os-release文件内容

	```Bash
	NAME="CentOS Linux"
	VERSION="7 (Core)"
	ID="centos"
	ID_LIKE="rhel fedora"
	VERSION_ID="7"
	PRETTY_NAME="CentOS Linux 7 (Core)"
	ANSI_COLOR="0;31"
	CPE_NAME="cpe:/o:centos:centos:7"
	HOME_URL="https://www.centos.org/"
	BUG_REPORT_URL="https://bugs.centos.org/"
	 
	CENTOS_MANTISBT_PROJECT="CentOS-7"
	CENTOS_MANTISBT_PROJECT_VERSION="7"
	REDHAT_SUPPORT_PRODUCT="centos"
	REDHAT_SUPPORT_PRODUCT_VERSION="7"
	```
	clair利用针对centos的解析器逐行解析该文件，提取ID以及VERSION_ID字段，最终把centos:7作为clair中的一个namespace概念。

- 探测已安装的软件包，上一步已经探测到了操作系统，自然可以知道系统的软件管理包是rpm还是dpkg。

	```
	debian, ubuntu : dpkg
	centos, rhel, fedora, amzn, ol, oracle : rpm
	```
	centos系统的软件管理包是rpm，而debain系统的软件管理是dpkg，获取软件管理工具之后，则可以在操作系统对应的路径下获取当前系统已安装了哪些包。

	```
	rpm：var/lib/rpm/Packages
	dpkg：var/lib/dpkg/status
	apk：lib/apk/db/installed
	```

	比如debian系统，从文件/var/lib/dpkg/status文件则可以探测到当前系统安装了哪些版本的软件，安装文件清单内容格式如下，由于篇幅有限，只截取了两个安装包展示。

	```Bash
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
	Clair又采取指定的解析器，对该文件逐行解析，提取Package以及Version字段，最终获取 libsmartcols1 2.29-1+deb9u1以及sed 4.4-1等重要属性信息。

- 数据持久化，把上面探测到的系统版本、以及系统上安装的各种软件包版本都存入数据库。Clair系统已经获取了各linux版本操作系统软件版本，以及对应软件版本存在的CVE。官方公布了某个软件在某个版本修复了哪个CVE，Clair只需要将当前镜像中的软件的版本与官方公布的版本进行比较。比如官方维护的CVE信息中公布了nginx 1.13.1修复了漏洞CVE-2015-10203，那么当前镜像中包含的版本为1.12.1的nginx必然存在漏洞CVE-2015-10203，这些版本比较都是基于同一个版本的操作系统之上比较的。

harbor页面具体的漏洞详细数据展示，还是通过UI系统调用Clair系统实时查询。

![harbor扫描详情](/blog/harbor_image_scan/003.jpg)

综上，我们可以完全扩展一下clair，并不局限于只扫描镜像，它扫描镜像的原理无非是提取了操作系统的安装清单，然后进行信息比对。
