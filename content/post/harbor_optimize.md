---
title: "Harbor镜像构建流程优化"
date: 2019-05-21T10:49:15+08:00
description: "详细介绍优化Harbor官方的镜像构建过程，提高镜像的构建速度"
Tags:
- harbor
- docker
Categories:
- docker
---

以Harbor v1.4~v1.6之间的版本为例，详细分析其构建离线包的整个流程，然后结合当前的环境以及目标，去优化这个构建流程。
在Harbor的整个构建流程中，涉及的技术知识点包括Shell、Python、Make(Makefile)、Go、Docker等，不要求对这些技术熟练精通，
能看懂其中的相关命令即可。

Make是Linux GNU中的一个命令工具，用于解释Makefile中的指令规则，开源项目(distribution、helm)通常用Makefile来描述了整个项目所有文件的编译顺序、编译规则。
Harbor也不例外，使用两个Makefile来组织整个项目的编译构建工作。

官方构建离线包的命令

```bash
make package_offline GOBUILDIMAGE=golang:1.7.3 COMPILETAG=compile_golangimage \
CLARITYIMAGE=vmware/harbor-clarity-ui-builder:1.2.7 CLAIRFLAG=true
```

该make命令是基于项目根目录harbor下执行，一个make命令将整个构建流程组织起来，这个过程中又嵌入了子Makefile的执行。

```bash
$GOPATH/src/github.com/vmware/harbor/Makefile
$GOPATH/src/github.com/vmware/harbor/make/photon/Makefile
```

在国内宿主机上构建命令成功执行一次的时间应该至少在15分钟左右，尤其是在机器上第一次构建耗时更长，如此耗时，很有必要去了解一下编译构建的逻辑原理。

- 执行构建命令的宿主机能访问外网，还必须要能fanqian
- 部分镜像的构建上下文范围过大，导致发送给Docker Daemon的文件数据过大
- 部分镜像并未变动，重复构建
- 依赖的第三方项目(如Clair、Registry)，从Google云下载二进制包或者github拉取源代码构建，速度过慢
- 许多模块镜像具备很多相同的指令，未抽取做成通用的基础镜像


首先看一下Makefile中ui、jobservice、Adminserver源码的编译，以ui为例：

```bash
@echo "compiling binary for ui (golang image)..."
@echo $(GOBASEPATH)
@echo $(GOBUILDPATH)
@$(DOCKERCMD) run --rm -v $(BUILDPATH):$(GOBUILDPATH) -w $(GOBUILDPATH_UI) $(GOBUILDIMAGE) \
$(GOIMAGEBUILD) -v -o $(GOBUILDMAKEPATH_UI)/$(UIBINARYNAME)
@echo "Done."
```

假如当前$GOPATH=/data/go_workspace，翻译之后的编译命令如下

```docker
docker run --rm -v /data/go_workspace/src/github.com/vmware/harbor:/go/src/github.com/vmware/harbor \
-w /go/src/github.com/vmware/harbor/ui golang:1.7.3 /usr/local/go/bin/go build -v -o \
/go/src/github.com/vmware/harbor/make/dev/ui/harbor_ui　
```

上面命令关联的几个目录如下

```bash
A=/go/src/github.com/vmware/harbor
B=/go/src/github.com/vmware/harbor/ui
C=/go/src/github.com/vmware/harbor/make/dev/ui/
D=/data/go_workspace/src/github.com/vmware/harbor/make/dev/ui/harbor_ui
```

上面编译ui模块源码时，首先启动了一个容器，该容器运行的镜像就是golang:1.7.3，镜像tag也就是make命令中`GOBUILDIMAGE`指定的值。容器启动时，将宿主机上的
源码目录harbor挂载到容器`A`目录，同时利用`-w`设定工作目录，编译ui时将工作目录设置为`B`，然后执行go build编译，将编译后生成的可执行文件harbor_ui输出到C目录，
即宿主机目录`D`，go build命令退出时，则自动将volume删除，Adminserver、Jobservice源码编译也是这个过程，最后分别生成harbor-adminserver、harbor-jobservice，
输出到各自对应的目录中。

首次编译时，需要去拉取指定tag的golang镜像，这个镜像非常大，golang:1.7.3就将近700M，编译每个模块源码时都会先启动一个Docker容器，这样的优点不依赖宿主机
的go环境，直接在容器中处理，缺点就是占用资源大，非常耗时。我们部署Harbor的机器，或者专门用来构建打包的机器肯定已经具备了go语言的编译环境，根本无须这种方式，
直接基于宿主机设置的$GOPATH进行即可。

```bash
harbor_project="$GOPATH/src/github.com/vmware/harbor"
echo "compiling binary for ui"
cd $harbor_project/src/ui
go build -v -o ../../deploy/docker/ui/harbor_ui
echo "Done"
```

编译源码生成可执行文件之后，看下其构建镜像过程，构建镜像过程都在子Makefile文件中。

```Makefile
_build_ui:	
	@echo "building ui container for photon..."
	$(DOCKERBUILD) -f $(DOCKERFILEPATH_UI)/$(DOCKERFILENAME_UI) -t $(DOCKERIMAGENAME_UI):$(VERSIONTAG) .
	@echo "Done."
```

$GOPATH=/data/go_workspace/，翻译出来的命令为

```go
docker build -f $GOPATH/src/github.com/vmware/harbor/make/photon/Dockerfile \
-t vmware/harbor-ui:dev . 
```

上面docker build的构建上下文是当前根目录harbor，实际上构建UI镜像时需要的文件仅仅为可执行文件harbor_ui、脚本start.sh以及Dockerfile，
无须将harbor目录下所有的文件发送给Docker Daemon，至少1GB以上，只需在构建镜像时新建一个子目录，将镜像构建需要的文件全部复制到此目录，进行构建即可。

```docker
docker build -f deploy/docker/ui/Dockerfile -t vmware/harbor-ui:$image_tag deploy/docker/ui
```

Harbor依赖的第三方组件包括Docker Registry、CoreOS的Clair、Helm Chartmuseum，构建这些组件镜像时都需要去外网获取二进制文件或者源代码。Registry与Clair都需要从
google上下载官方打好的二进制包，而Chartmuseum基于github项目地址拉取源码本地构建，非常耗时，尤其国内网络，访问google超慢。

```Makefile
_build_clair:
	@if [ "$(CLAIRFLAG)" = "true" ] ; then \
        if [ "$(BUILDBIN)" != "true" ] ; then \
            rm -rf $(DOCKERFILEPATH_CLAIR)/binary && mkdir -p $(DOCKERFILEPATH_CLAIR)/binary && \
            $(call _get_binary, https://storage.googleapis.com/harbor-builds/bin/clair/release2.0-$(CLAIRVERSION)/clair, $(DOCKERFILEPATH_CLAIR)/binary/clair); \
        else \
            cd $(DOCKERFILEPATH_CLAIR) && $(DOCKERFILEPATH_CLAIR)/builder $(CLAIRVERSION) && cd - ; \
        fi ; \
        echo "building clair container for photon..." ; \
        $(DOCKERBUILD) -f $(DOCKERFILEPATH_CLAIR)/$(DOCKERFILENAME_CLAIR) -t $(DOCKERIMAGENAME_CLAIR):$(CLAIRVERSION)-$(VERSIONTAG) . ; \
        rm -rf $(DOCKERFILEPATH_CLAIR)/binary; \
        echo "Done." ; \
    fi
```
这些依赖的第三方组件，我们可以自行通过github源码构建镜像，上传到我们自己的仓库中，将这些镜像的构建过程从Makefile中剔除。

在harbor这个项目中，我们基本不会去修改自带的web ui页面模块，构建镜像时不用去生成前端js组件，把上一次生成的文件copy出来，当成源代码保存在harbor目录中即可，先
看一下Makefile中是如何构建前端js模块的。

```Makefile
compile_clarity:
	@echo "compiling binary for clarity ui..."
	@if [ "$(HTTPPROXY)" != "" ] ; then \
		$(DOCKERCMD) run --rm -v $(BUILDPATH)/src:$(CLARITYSEEDPATH) $(CLARITYIMAGE) $(SHELL) $(CLARITYBUILDSCRIPT) -p $(HTTPPROXY); \
	else \
		$(DOCKERCMD) run --rm -v $(BUILDPATH)/src:$(CLARITYSEEDPATH) $(CLARITYIMAGE) $(SHELL) $(CLARITYBUILDSCRIPT); \
	fi
	@echo "Done."
```

实际执行的命令如下

```docker
docker run --rm -v $GOPATH/src/github.com/vmware/harbor/src:/harbor_src \
vmware/harbor-clarity-ui-builder:1.2.7 /bin/bash /entrypoint.sh
```

启动一个运行harbor-clarity-ui-builder:1.2.7镜像的容器，执行entrypoint.sh脚本，主要采用的是angular构建打包，依赖包非常多，每次构建都需要从新下载，
构建成功之后会生成几个js、css文件。

```bash
build.min.js  
clarity-icons.min.css  
clarity-icons.min.js  
clarity-ui.min.css  c
ustom-elements.min.js 
mutationobserver.min.js 
styles.css
```
只需将以上文件作为源码保存在项目下，下次构建镜像时直接利用。

从各组件的Dockerfile指令看出，很多雷同，不需要每次都去下载那些依赖组件，可以在官方的基础上构建我们自己的基础镜像。
官方构建ui模块的Dockerfile

```dockerfile
FROM vmware/photon:1.0

RUN tdnf distro-sync -y || echo \
    && tdnf erase vim -y \
    && tdnf install sudo -y \
    && tdnf clean all \
    && groupadd -r -g 10000 harbor && useradd --no-log-init -r -g 10000 -u 10000 harbor \
    && mkdir /harbor/

HEALTHCHECK CMD curl -s -o /dev/null -w "%{http_code}" 127.0.0.1:8080/api/systeminfo|grep 200
COPY ./make/dev/ui/harbor_ui ./src/favicon.ico ./make/photon/ui/start.sh ./VERSION /harbor/
COPY ./src/ui/views /harbor/views
COPY ./src/ui/static /harbor/static

RUN chmod u+x /harbor/start.sh /harbor/harbor_ui
WORKDIR /harbor/
    
ENTRYPOINT ["/harbor/start.sh"]
```
将安装vim、sudo等提取出来，做成基础镜像Dockerfile。

```Dockerfile
FROM vmware/photon:1.0

MAINTAINER sunjinfu@163.com

RUN tdnf distro-sync -y || echo \
    && tdnf erase vim -y \
    && tdnf install sudo -y \
    && tdnf clean all \
    && groupadd -r -g 10000 harbor && useradd --no-log-init -r -g 10000 -u 10000 harbor \
    && mkdir /harbor/
```

构建基础镜像

```Docker
docker build -t xx.com/xx/harbor-photon:1.0
```
然后在基础镜像基础上构建UI、Adminserver、Jobservice等镜像

```Dockerfile
FROM xx.com/xx/harbor-photon:1.0
 
MAINTAINER sunjinfu@163.com
 
HEALTHCHECK CMD curl -s -o /dev/null -w "%{http_code}" 127.0.0.1:8080/api/systeminfo|grep 200
COPY harbor_ui favicon.ico start.sh VERSION /harbor/
COPY views /harbor/views
COPY static /harbor/static
 
RUN chmod u+x /harbor/start.sh /harbor/harbor_ui
WORKDIR /harbor/
 
ENTRYPOINT ["/harbor/start.sh"]
```

至此，整个构建流程差不多了，我们构建的镜像不再打包，直接上传到自己的镜像仓库即可，一个完整的构建shell脚本如下。

```bash
#!/bin/bash

#author: sunjinfu
#compile go binary, go version must be at least 1.7.3

set -e
if [ -n $GOPATH ]
then
    echo "GOPATH is $GOPATH"
else
    echo "env GOPATH lost"
    exit 1
fi
harbor_project="$GOPATH/src/github.com/vmware/harbor"
echo "harbor project root directory is $harbor_project"
if [ ! -d $harbor_project ]
then
    echo "harbor project not exist"
    exit 1
else
    echo ""
fi
echo "deploy args:" $*
image_tag=$1
if [ ! -n "$image_tag" ]
then
    echo "docker image tag is empty"
    exit 1
else
    echo "docker image tag is $image_tag"
fi
echo "compiling binary for adminserver"
cd $harbor_project/src/adminserver
go build -v -o ../../deploy/docker/adminserver/harbor_adminserver
echo "Done"
echo "compiling binary for jobservice"
cd $harbor_project/src/jobservice
go build -v -o ../../deploy/docker/jobservice/harbor_jobservice
echo "Done"
echo "compiling binary for ui"
cd $harbor_project/src/ui
go build -v -o ../../deploy/docker/ui/harbor_ui
echo "Done"

cd $harbor_project
echo "docker building adminserver image"
docker build -f deploy/docker/adminserver/Dockerfile -t xx.com/container/harbor-adminserver:$image_tag deploy/docker/adminserver
echo "Done"
echo "docker building jobservice image"
docker build -f deploy/docker/jobservice/Dockerfile -t xx.com/container/harbor-jobservice:$image_tag deploy/docker/jobservice
echo "Done"
echo "docker building ui image"
cp -r src/ui/static deploy/docker/ui
cp -r src/ui/views deploy/docker/ui
cp src/favicon.ico deploy/docker/ui
cp VERSION deploy/docker/ui
docker build -f deploy/docker/ui/Dockerfile -t xx.com/container/harbor-ui:$image_tag deploy/docker/ui
echo "Done"

docker login xx.com -u root -p 1234567890

docker push xx.com/container/harbor-adminserver:$image_tag
docker push xx.com/container/harbor-jobservice:$image_tag
docker push xx.com/container/harbor-ui:$image_tag
echo "docker images push done"

echo "cleaning start"
rm -rf deploy/docker/adminserver/harbor_adminserver
rm -rf deploy/docker/jobservice/harbor_jobservice
rm -rf deploy/docker/ui/harbor_ui
#docker rmi xx.com/container/harbor-adminserver:$image_tag
#docker rmi xx.com/container/harbor-jobservice:$image_tag
#docker rmi xx.com/container/harbor-ui:$image_tag
rm -rf deploy/docker/ui/views
rm -rf deploy/docker/ui/static
rm -rf deploy/docker/ui/VERSION
rm -rf deploy/docker/ui/favicon.ico
echo "Done"
```

基于已有仓库镜像打harbor离线安装包脚本

```bash
#!/bin/bash

#author: sunjinfu

set -e
echo "before execute this script, you should use correct user and login xx.com successfully."

echo "deploy args:" $*
image_tag=$1
if [ ! -n "$image_tag" ]
then
    echo "docker image tag is empty"
    exit 1
else
    echo "docker image tag is $image_tag"
fi
docker pull xx.com/container/harbor-config:$image_tag

container_id="`docker create --name harbor-config xx.com/container/harbor-config:$image_tag`"
if [ ! -n "$container_id" ]
then
    echo "docker container harbor-config create failed"
    exit 1
else
    echo "docker container harbor-config id => $container_id"
fi
volume_line=`docker inspect $container_id | grep "/docker/volumes/"`
echo "volume line => $volume_line"
volume_line_tmp=${volume_line#*/}
volume_dir=/${volume_line_tmp:start:length-2}
echo "harbor-config container volume dir => $volume_dir"
if [ ! -n "$volume_dir" ]
then
    echo "docker container harbor-config volume dir is empty"
    exit 1
else
    echo "docker container harbor-config volume dir => $volume_dir"
fi

harbor_package=harbor-offline-package
mkdir $harbor_package
cd $harbor_package
cp -r $volume_dir/harbor .
echo "current dir => `pwd`"
echo "pulling harbor docker images ..."
harbor_log="xx.com/container/harbor-log:stable"
harbor_nginx="xx.com/container/harbor-nginx:stable"
harbor_registry="xx.com/container/registry:latest"
harbor_postgresql="xx.com/container/postgresql:latest"
harbor_clair="xx.com/container/clair:v2.0.6"
harbor_adminserver="xx.com/container/harbor-adminserver:$image_tag"
harbor_ui="xx.com/container/harbor-ui:$image_tag"
harbor_jobservice="xx.com/container/harbor-jobservice:$image_tag"

docker pull $harbor_log
docker pull $harbor_nginx
docker pull $harbor_registry
docker pull $harbor_postgresql
docker pull $harbor_clair

docker pull $harbor_adminserver
docker pull $harbor_ui
docker pull $harbor_jobservice

echo "Done"

harbor_image_filename="harbor.image_$image_tag.tar"

echo "saving harbor docker images..."
docker save $harbor_log $harbor_nginx $harbor_registry $harbor_postgresql $harbor_clair $harbor_adminserver \
        $harbor_ui $harbor_jobservice > $harbor_image_filename
gzip $harbor_image_filename
echo "harbor docker images save successfully"

echo "building offline install package..."

harbor_image_gz="$harbor_image_filename.gz"

mv $harbor_image_filename.gz harbor

sed -i "s/\${version}/$image_tag/" harbor/docker-compose.yml

tar -zcvf harbor-offline-installer-$image_tag.tgz harbor

echo "Done"

rm -rf harbor
docker rm -v $container_id

echo "harbor offline install package has build successfully in `pwd`"
echo "you can copy harbor-offline-installer-$image_tag.tgz to other machine and install it!"
```

离线包安装脚本

```bash
#!/bin/bash

#author: sunjinfu

set -e
file_num=`ls -lrt | grep 'harbor-offline-installer-*.*.tgz'| wc -l`
if [ $file_num -eq 0 ]
then
    echo "current directory can not find any harbor offline install package"
    exit 1
else
    if [ $file_num -eq 1 ]
    then
        echo "find harbor offline install package"
        ls | grep 'harbor-offline-installer-*.*.tgz'
    else
        echo "find multi harbor offline install packages, please delete redundant packages"
        exit 1
    fi
fi

echo "start tar harbor offline install package"
tar -xvf harbor-offline-installer-*.*.tgz
echo "Done"

echo "load harbor docker images, please wait some seconds..."
docker load -i harbor.image_*.*.tar.gz
echo "load harbor docker images successfully."

chmod u+x harbor/prepare harbor/install.sh

echo "================================================================================================"
echo "#harbor has init successfully, please enter in `pwd`/harbor to modify harbor.cfg."
echo "#when you complete harbor.cfg, please exec [./prepare] first, then [./install.sh] to install harbor."
echo "#if need image scan, please exec [./prepare --with-clair] first, then [./install.sh --with-clair]."
echo "================================================================================================"
```