---
title: "Dockerfile详解以及高级技巧"
thumbnail: "img/docker.jpg"
date: 2019-04-07T20:35:36+08:00
description: "详细介绍Dockerfile的基本指令以及构建Docker镜像的高级技巧知识"
Tags:
- docker
Categories:
- docker
---
Docker是通过读取Dockerfile文件来自动构建镜像，Dockerfile其实就是一个包含了很多命令行指令的文本文件，通过这些指令来装配一个镜像。
要掌握Docker构建镜像的技巧，就必须首先了解Dockerfile的基本指令，下面先详细介绍Dockerfile中的一些常用指令。

### 基本指令

- **FROM**

    `From`用来在构建镜像时指定一个基础镜像，一个有效的Dockerfile文件必须以`From`指令开始，可以通过`AS name`命令给当前这个创建的阶段一个别名，
    然后这个别名在后序的`From`指令以及`COPY --from=name`可以引用。当然，严格的来说Dockerfile并不是必须以`From`指令开始，因为`From`
    指令也支持变量，在`From`之前可以通过`ARG`定义变量，这个`ARG`必须在第一个`From`指令之前。
    ```Dockerfile
    ARG  CODE_VERSION=latest
    FROM base:${CODE_VERSION}
    CMD  /code/run-app

    FROM extras:${CODE_VERSION}
    CMD  /code/run-extras
    ```
- **MAINTAINER**

    这个指令用来设置创建镜像的作者信息，已经废弃，官方提倡使用更灵活的`LABEL`指令，`LABEL`指令可以设置更多的属性。
    
    ```Dockerfile
    MAINTAINER sunjinfu@163.com
    ```
    通过`LABEL`指令代替`MAINTAINER`
    
    ```Dockerfile
    LABEL maintainer="sunjinfu@163.com"
    ```
- **LABEL**
      
      `LABEL`指令主要用来给镜像增加一些metadata，一个LABEL是一个key-value形式的键值对，如果LABEL中需要包含空格或者反斜杠，必须用双引号括起来。
      ```
      LABEL "name"="sunjinfu"
      LABEL email="sunjinfu@163.com"
      LABEL version="1.0"
      LABEL description="I am one \
            good man."
      ```
      
      一个镜像可能有很多`LABEL`，可以通过以下两种方式尽量定义在一行中。
      
      ```
      LABEL "name"="sunjinfu" email="sunjinfu@163.com"
      ```
      
      ```
      LABEL "name"="sunjinfu" \
            email="sunjinfu@163.com" \
            age="20"
      ```
      `LABEL`是可以从base镜像那继承的，如果有冲突，一般先前定义的`LABEL`都会被覆盖，镜像具有哪些`LABEL`，通过`docker inspect`命令即可查看。
      
- **ENV**

    `ENV`指令用来设置环境变量，在构建镜像阶段，后续所有的指令都可以使用它，设置环境变量有两种方式。
    
    > * ENV \<key\> \<value\>
    > * ENV \<key\>=\<value\> ...
    
    `ENV <key> <value>`只能设置单个变量，而`ENV <key>=<value>`可以同时设置多个，可以使用`"`或者`\`包含空格。
    
    ```
    ENV name="sun jinfu" email=sunjinfu@163.com\ sunjinfu@126.com\ sunjinfu@gmail.com \
        address=beijing
    ```
    
    等价于
    
    ```
    ENV name sun jinfu
    ENV email sunjinfu@163.com sunjinfu@126.com sunjinfu@gmail.com
    ENV address beijing
    ```

- **VOLUME**

	`VOLUME`指令用来创建一个给定名字的挂载点，如 `VOLUME ["/data"]` ，当容器运行的时候，可以很方便的将容器目录中的数据与主机目录数据共享。`VOLUME`
	是以JSON数组形式解析的，因此必须以`"`括起来，如 `VOLUME ["/data", "/var/log"]`
	
- **WORKDIR**

	`WORKDIR`用于设置工作目录，如果`WORKDIR`指令设置的目录不存在则会自动创建，在一个Dockerfile文件中可以通过`WORKDIR`设置多次工作目录，如：
	
	```Dockerfile
	WORKDIR /a
	WORKDIR b
	WORKDIR c
	RUN pwd
	```

	`pwd`命令的输出结果是/a/b/c，`WORKDIR`指令可以查找在它之前通过`ENV`指令设置的环境变量。
	
	```Dockerfile
	ENV DIRPATH /path
	WORKDIR $DIRPATH/$DIRNAME
	RUN pwd
	```
	
	`pwd`命令的输出结果是/path/$DIRNAME
	
- **EXPOSE**

	`EXPOSE`指令用于通知Docker当前容器在运行时的监听端口，协议支持`TCP`、`UDP`，默认`TCP`，用法:
	
	```Dockerfile
	EXPOSE 80/tcp
	EXPOSE 80/udp
	EXPOSE 3306
	```
	`EXPOSE`指令暴露容器端口后，执行`docker run`命令时带上flag大写 `-P`即可将容器暴露端口映射到主机的随机端口(49000~49900)，可以通过flag小写`-p`指定
	端口映射，此时实际`EXPOSE`指令并未发生任何作用，被覆盖。
	
- **HEALTHCHECK**

	`HEALTHCHECK`指令有两种方式:
	
	> * HEALTHCHECK \[OPTIONS] CMD command  #通过在容器内部运行一个命令健康检查
	> * HEALTHCHECK NONE #禁用从基础镜像那继承任何健康检查
	
	`HEALTHCHECK`指令告诉Docker如何检查容器是否正常，当给一个容器定义了一个健康检查规则时，那么容器则有一个健康状态。当健康检查通过时，容器则会展示`healthy`，
	否则展示为`unhealthy`，`HEALTHCHECK`的OPTIONS参数如下:
	
	> * --interval=DURATION (default: 30s)
	> * --timeout=DURATION (default: 30s)
	> * --start-period=DURATION (default: 0s)
	> * --retries=N (default: 3)
	
	- 健康检查会在容器启动之后的`interval`秒首次执行，之后间隔`interval`秒进行健康检查，注意这里的容器启动，并不是容器内部的应用启动，比如在容器中部署了一个tomcat
		应用，这个tomcat应用需要50秒才能完成启动，而容器启动只需2秒，如果`interval`设置为30秒，健康检查又设置为调用容器内部应用的一个接口，
		每次健康检查都返回非200状态码，这样Docker就不断的重启该容器，陷入无限循环了，正确的做法是将`interval`设置大一点，比如60秒。
	- `timeout`健康检查超时时间
	- `start-period` 容器启动初始化时间，在这段时间内如果健康检查失败，并不是累加到`retries`字段上。
	
	健康检查的`CMD`指令返回状态码，`0`表示容器健康状态，`1`表示容器不健康状态，`2`保留状态码，暂未使用。下面这个例子，http请求的响应状态码是401才表示系统健康状态。
	
	```Dockerfile
	HEALTHCHECK CMD curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8080/api/jobs/replication/1/log|grep 401
	```
	
- **ADD**

	`ADD`指令有两种方式，第二种方式可以支持包含空格的路径，因为其用`"`括起来。
	
	
	> * ADD \[--chown=\<user\>:\<group\>] \<src\>... \<dest\>
	> * ADD \[--chown=\<user\>:\<group\>] ["\<src\>",... "\<dest\>"]

	`ADD`指令用于将主机中文件、目录或者通过链接指定的远程文件复制到镜像中的文件系统中，要复制的源文件、源目录都是相对于当前构建镜像的上下文。
	`src`可以包括一些匹配表达式，如：
	
	```Dockerfile
	ADD hom* /mydir/        # 将上下文目录中所有文件名以`hom`开始的文件复制到镜像的/mydir/目录中
	ADD hom?.txt /mydir/    # ? 匹配单个字符
	```
	`dest`可以是一个绝对路径，也可以是一个基于`WORKDIR`的相对路径。
	
	```Dockerfile
	ADD test relativeDir/          # adds "test" to `WORKDIR`/relativeDir/
	ADD test /absoluteDir/         # adds "test" to /absoluteDir/
	```
	所有复制到镜像中的文件或者目录，默认都是以UID、GID为0的用户创建，除非通过`--chown`指定，注意，如果容器文件系统中没有`/etc/passwd`或者`/etc/group`，
	或者通过`--chown`指定的用户信息不存在，那么`ADD`操作则会失败。
	
	```Dockerfile
	ADD --chown=55:mygroup files* /somedir/
	ADD --chown=bin files* /somedir/
	```
	
	`ADD`指令的源文件都是基于构建上下文，所以不允许`ADD ../somefile /somefile`，只能向下不能向上，添加多个文件可以写在一行。
	
	```Dockerfile
	ADD start.sh harbor_jobservice /harbor/   
	```
	
	也可以用`\`写在多行
	
	```Dockerfile
	ADD docker-compose.clair.yml \
    docker-compose.yml \
    harbor.cfg \
    install.sh \
    registry.sql \
    prepare \
    /data/harbor/
	```
	
	添加整个目录到镜像层中
	
	```Dockerfile
	ADD /data/harbor   /harbor
	```
	
	`ADD`指令会自动将`gzip`、`tar.gz`等压缩包自动解压，当然文件是否被解压不是根据文件名决定的，而是文件内容，即使有一个空的文件以`.tar.gz`结尾，
	它也不会被解压，只是简单的将该文件添加。
	
	注意:
	
	> * src有多个，那么dest必须是个目录，dest必须以`/`结尾
	> * dest没有以`/`结尾，它会被认为是个常规的文件，直接将src文件的内容写到dest这个文件中
	> * dest目录中任何一级目录不存在，都会被自动创建 
	
- **COPY**

	`COPY`指令与`ADD`指令功能相似，也支持两种方式。
	
	> * COPY \[--chown=\<user\>:\<group\>] \<src\>... \<dest\>
	> * COPY \[--chown=\<user\>:\<group\>] ["\<src\>",... "\<dest\>"]

	`COPY`指令与`ADD`指令的区别就是它不支持远程URL文件复制，同时压缩包不会自动解压，其他用法基本与`ADD`指令一致。
	
- **USER**

	`USER`指令用于设置容器运行时的用户名、用户组，在Dockerfile中指定用户后，后续的`RUN`、`CMD`等指令，都将以该用户身份运行。
	
	```bash
		USER user
　　USER user:group
　　USER uid
　　USER uid:gid
	```
	
- **RUN**

    `RUN`指令有两种方式:
	
    > * RUN \<command>，命令用shell方式运行，在Linux中默认的shell命令`/bin/sh -c`，windows则是`cmd /S /C`
    > * RUN \["executable", "param1", "param2"] (exec方式)
    
    `RUN`指令主要用来在当前镜像的最上层执行一些命令继而生成新的一个镜像层，用shell方式执行命令时可以通过一个`\`继续在文档的下一行编写一个命令。
    
    ```bash
    RUN /bin/bash -c 'source $HOME/.bashrc; \
    echo $HOME'
    ```
    
    写在一行也完全可以
    
    ```bash
    RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
    ```
    
    如果不想用/bin/sh，想用/bin/bash执行命令，那么就不得不使用exec方式了。
    
    ```
    RUN ["/bin/bash", "-c", "echo hello"]
    ```
    
    exec方式被解析成JSON数组，因此每一项都必须使用`"`括起来，exec方式不会调用shell命令，因此`RUN [ "echo", "$HOME" ]`中的变量$HOME不会被替换，
    要么使用shell方式运行或者`RUN [ "sh", "-c", "echo $HOME" ]` exec方式即可。

- **CMD**

	`CMD`指令有三种方式:
	
	> * CMD \["executable","param1","param2"] (exec方式，推荐)
	> * CMD \["param1","param2"] (将param1、param2作为默认参数传递给ENTRYPOINT)
	> * CMD command param1 param2 (shell方式)
	
	Dockerfile文件中只有一个`CMD`指令会生效，如果你提供多个`CMD`，只有最后一个生效。`CMD`是容器启动时执行的一种默认行为，
	通过`docker run`运行容器时设置的命令会直接覆盖`CMD`，完全可以不设置`CMD`，设置`ENTRRYPOINT`即可。
	
- **ENTRYPOINT**

	`ENTRYPOINT`也有两种方式:
	
	> * ENTRYPOINT \["executable", "param1", "param2"] (exec，推荐)
	
	> * ENTRYPOINT command param1 param2 (shell)
	
	可以通过`ENTRYPOINT`将容器配置成可执行程序，通过`docker run`运行容器时的参数均可以传递给以exec方式执行的`ENTRYPOINT`指令上，它的默认参数
	可以通过`CMD`指令进行设定，`docker run`命令设置的参数可以直接覆盖`CMD`指令为`ENTRRYPOINT`设置的默认参数，执行`docker run <image> -d`命令时，
	参数 `-d`会直接传递给`ENTRRYPOINT`，当然通过 `docker run --entrypoint`可以覆盖Dockerfile中的`ENTRYPOINT`。
	
    shell执行方式会阻止`CMD`以及`docker run`等命令的任何参数，使用这种执行方式时，`ENTRYPOINT`相当于是`/bin/sh -c`的一个子命令，该子命令不能接受信号。
	这就意味着`ENTRYPOINT`指令设定的可执行程序在容器中的PID将不是1，将不会接受Unix的任何信号，即执行`docker stop <container>`命令时，可执行程序无法接受到
	`SIGTERM`。
	
	下面举个例子，首先编写一个Dockerfile文件，然后通过该Dockerfile构建镜像test，注意`CMD`、`ENTRYPOINT`都是采用了推荐的exec方式。
	
	```Dockerfile
	From centos
	ENTRYPOINT ["top", "-b"]
	CMD ["-c"]
	```
	
	构建镜像
	
	```docker
	#docker build -t test -f Dockerfile .
	Sending build context to Docker daemon  2.048kB
	Step 1/3 : From centos
	 ---> 9f38484d220f
	Step 2/3 : ENTRYPOINT ["top", "-b"]
	 ---> Running in f634088947be
	Removing intermediate container f634088947be
	 ---> 02dd3aeda4d3
	Step 3/3 : CMD ["-c"]
	 ---> Running in c90a93cf729d
	Removing intermediate container c90a93cf729d
	 ---> 7fae0be844ef
	Successfully built 7fae0be844ef
	Successfully tagged test:latest
	```
	
	运行容器
	
	```bash
	# docker run --rm --name test test:latest
	top - 02:46:03 up 55 days, 17:53,  0 users,  load average: 0.71, 0.24, 0.16
	Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
	%Cpu(s):  3.3 us,  6.7 sy,  0.0 ni, 90.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
	KiB Mem :  3882020 total,   561132 free,   560420 used,  2760468 buff/cache
	KiB Swap:        0 total,        0 free,        0 used.  2816488 avail Mem 

	PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    1 root      20   0   56060   1880   1444 R   0.0  0.0   0:00.03 top -b -c
	```
	
	容器中PID为1的进程为top命令，`CMD`指令将参数`-c`传递给了`ENTRYPOINT`。
	
### 构建技巧

- **上下文**

	在了解构建镜像上下文的概念之前，首先要了解清楚Docker的软件架构，最基础的就是Docker Client与Docker Daemon。
	Docker Daemon是Docker架构中的主体部分，具备服务端的功能，能直接接收Docker Client发起的请求。Docker Client发起的
	相关命令`docker pull`、`docker build`等都是请求Docker Daemon服务端。
	
	![docker](/blog/dockerfile/001.png)
	
	从上图中，你应该已经了解了，执行`docker build`命令构建镜像实际是在服务端进行的，只不过这个服务端运行在本地主机上，通过
	`docker version`命令可以查看版本。
	
	Docker构建镜像时有一个上下文的概念，执行`docker build`命令时首先会把上下文目录中的所有文件全部打包发送到Docker Daemon服务端，
	所以上下文目录的大小很大程度决定了你本次镜像构建的速度，这就是为什么不要直接把Linux的根目录作为构建上下文的原因，
	构建镜像最好的习惯是新建一个目录作为构建的上下文，把Dockerfile中需要的文件都复制到该目录，然后执行命令构建镜像。
	
	下面举个例子，编写Dockerfile，这个Dockerfile只与主机中的一个readme.txt文件有关。
	
	```Dockerfile
	From centos
	ADD readme.txt /data
	ENTRYPOINT ["top", "-b"]
	CMD ["-c"]
	```
	
	当前Dockerfile文件、readme.txt均在/root目录下，该目录下还有其他一些与镜像无关的文件。
	
	```bash
	[root@vm ~]# ls -lrt
	total 372568
	-rw-r--r-- 1 root root 127163815 Aug 25  2018 go1.11.linux-amd64.tar.gz
	-rw-r--r-- 1 root root      2672 Feb 13 17:04 extranet.sh
	-rw-r--r-- 1 root root 127163815 Apr 10 13:17 a.gz
	-rw-r--r-- 1 root root 127163815 Apr 10 13:18 b.gz
	drwxr-xr-x 2 root root      4096 Apr 10 13:19 tmp
	-rw-r--r-- 1 root root         8 Apr 10 13:20 readme.txt
	-rw-r--r-- 1 root root        69 Apr 10 13:21 Dockerfile
	```
	
	构建镜像
	
	```docker
	[root@vm ~]# docker build -t readme:v1 -f Dockerfile .
	Sending build context to Docker daemon  763.8MB
	Step 1/4 : From centos
	 ---> 9f38484d220f
	Step 2/4 : ADD readme.txt /data
	 ---> 90f4d2c008f2
	Step 3/4 : ENTRYPOINT ["top", "-b"]
	 ---> Running in 6a07e260807c
	Removing intermediate container 6a07e260807c
	 ---> 927d6cebf38d
	Step 4/4 : CMD ["-c"]
	 ---> Running in 594773b76eab
	Removing intermediate container 594773b76eab
	 ---> 6917f27020e0
	Successfully built 6917f27020e0
	Successfully tagged readme:v1
	You have new mail in /var/spool/mail/root
	```
	
	从上面Docker输出信息`Sending build context to Docker daemon  763.8MB`，一共发送了763.8M文件到Docker Deamon，而Dockerfile只需要一个readme.txt文件，这就是
	构建上下文没有正确选择，在/root目录下新建一个文件夹docker作为上下文，把readme.txt都移到docker文件夹中，进行构建。
	
	```docker
	[root@vm172-20-0-15 ~]# docker build -t readme:v2 -f Dockerfile docker
	Sending build context to Docker daemon  2.607kB
	Step 1/4 : From centos
	---> 9f38484d220f
	Step 2/4 : ADD readme.txt /data
	---> Using cache
	---> 90f4d2c008f2
	Step 3/4 : ENTRYPOINT ["top", "-b"]
	---> Using cache
	---> 927d6cebf38d
	Step 4/4 : CMD ["-c"]
	---> Using cache
	---> 6917f27020e0
	Successfully built 6917f27020e0
	Successfully tagged readme:v2
	```
	
	从上面这个输出信息看，Docker Client只向Docker Daemon服务端发送了`2.607kB`大小文件。通常在大型项目中，自动化构建镜像时，一定要注意上下文的作用与范围。
	
- **镜像大小**

	镜像太大，同时网络带宽又有限，那么通过`docker pull`命令从镜像仓库拉取镜像时非常耗时，所以优化镜像大小很有必要，可以从以下几方面优化。
	
	> * 选择合适的基础镜像
	> * 优化Dockerfile中的指令编写，同一个指令尽量写在一行
	> * 根据应用的开发语言，剥离相关的环境依赖，比如go运行时并不需要的编译环境
	
	下面以go项目container为例，选择用golang:alpine为基础镜像，这个基础镜像相比于golang:1.11.1又小很多，接着将container项目的整个源码目录复制到镜像层的GOPATH
	目录下的src目录，然后执行go install编译源码，链接成可执行文件container，Dockerfile文件如下:
	
	```Dockerfile
	FROM golang:alpine
	COPY src /go/src
	RUN go install -v container
	ENTRYPOINT ["/go/bin/container"]
	```
	
	构建镜像
	
	```docker
	[root@vm docker]# docker build -t container:v1.0 .
	Sending build context to Docker daemon  57.74MB
	Step 1/4 : FROM golang:alpine
	 ---> 20ff4d6283c0
	Step 2/4 : COPY src /go/src
	 ---> dd2d3480ebd0
	Step 3/4 : RUN go install -v container
	 ---> Running in d67a1cf74365
	 ---> b6c5ed0d75f5
	Removing intermediate container d67a1cf74365
	Step 4/4 : ENTRYPOINT /go/bin/container
	 ---> Running in 734b0fdc6e5c
	 ---> 395503e87bc1
	Removing intermediate container 734b0fdc6e5c
	Successfully built 395503e87bc1
	Successfully tagged container:v1.0
	```
	
	查看镜像大小
	
	```bash
	REPOSITORY    TAG                 IMAGE ID            CREATED             SIZE
	container     v1.0                395503e87bc1        2 minutes ago       386MB
	```
	
	区区一个简单的go项目竟然达到386M，并且整个项目源码也在容器中，不安全。下面将go的编译环境去除，因为go项目运行时不依赖go sdk相关组件。
	优化一下Dockerfile文件，将alpine作为最终的基础镜像。
	
	```Dockerfile
	FROM golang:alpine AS build-env
	MAINTAINER sunjinfu@163.com
	ADD src /go/src
	RUN go build container

	FROM alpine
	RUN mkdir /go
	WORKDIR /go
	COPY --from=build-env /go/container /go
	EXPOSE 8080
	ENTRYPOINT ["./container"]
	```
	
	构建镜像
	
	```docker
	[root@vm docker]# docker build -t container:v2.0 . 
	Sending build context to Docker daemon  57.74MB
	Step 1/10 : FROM golang:alpine AS build-env
	 ---> 20ff4d6283c0
	Step 2/10 : MAINTAINER sunjinfu@163.com
	 ---> Using cache
	 ---> ac5b51c8ee48
	Step 3/10 : ADD src /go/src
	 ---> a1b828a87e8d
	Step 4/10 : RUN go build container
	 ---> Running in 7f4c09d3e576
	 ---> cd073b46d45d
	Removing intermediate container 7f4c09d3e576
	Step 5/10 : FROM alpine
	 ---> 5cb3aa00f899
	Step 6/10 : RUN mkdir /go
	 ---> Running in 8a7bd2f9025d
	 ---> 05b4a219e3e5
	Removing intermediate container 8a7bd2f9025d
	Step 7/10 : WORKDIR /go
	 ---> fcb8526b7b76
	Removing intermediate container a8f531d742a7
	Step 8/10 : COPY --from=build-env /go/container /go
	 ---> 55df14427b9c
	Step 9/10 : EXPOSE 8080
	 ---> Running in 82f9e5752c90
	 ---> f5c9c6e4c1ed
	Removing intermediate container 82f9e5752c90
	Step 10/10 : ENTRYPOINT ./container
	 ---> Running in 9ccd355dd431
	 ---> 053388fa3e2c
	Removing intermediate container 9ccd355dd431
	Successfully built 053388fa3e2c
	Successfully tagged container:v2.0
	```
	
	查看镜像大小，container:v2.0版本的镜像只有15MB。
	
	```bash
	REPOSITORY        TAG            IMAGE ID            CREATED              SIZE
	container         v2.0           053388fa3e2c        About a minute ago   15.6MB
	```
	
- **镜像缓存**

	Docker会缓存已有镜像的镜像层，构建新镜像时，如果某个镜像层已经存在，则直接利用缓存的镜像层，无须重新创建。
	
	```Dockerfile
	From centos
	ADD readme.txt /data
	ENTRYPOINT ["top", "-b"]
	CMD ["-c"]
	```
	
	构建镜像 
	
	```docker
	[root@vm ~]# docker build -t test:v1.0 -f Dockerfile docker
	Sending build context to Docker daemon  2.629kB
	Step 1/4 : From centos
	 ---> 9f38484d220f
	Step 2/4 : ADD readme.txt /data
	 ---> a097ddb31783
	Step 3/4 : ENTRYPOINT ["top", "-b"]
	 ---> Running in 89a4e6cd646b
	Removing intermediate container 89a4e6cd646b
	 ---> d9a2db7afdf5
	Step 4/4 : CMD ["-c"]
	 ---> Running in e51982daa4bd
	Removing intermediate container e51982daa4bd
	 ---> 92dc03cdc871
	Successfully built 92dc03cdc871
	Successfully tagged test:v1.0
	```
	
	下面需要构建另外一个镜像，Dockerfile如下：
	
	```Dockerfile
	From centos
	ADD readme.txt /data
	EXPOSE 8080
	ADD service.yml /data
	ENTRYPOINT ["top", "-b"]
	```
	
	构建镜像，在构建test:v1.0镜像时，`ADD readme.txt /data`这一镜像层的id是`a097ddb31783`，再次构建test:v2.0时将直接利用该镜像层缓存，注意查看输出信息中的
	`Using cache`，当然如果不想让Docker利用缓存，可以带上Flag参数`--no-cache`重新构建。
	
	```docker
	[root@vm ~]# docker build -t test:v2.0 -f Dockerfile docker
	Sending build context to Docker daemon  4.143kB
	Step 1/5 : From centos
	 ---> 9f38484d220f
	Step 2/5 : ADD readme.txt /data
	 ---> Using cache
	 ---> a097ddb31783
	Step 3/5 : EXPOSE 8080
	 ---> Running in 6c845ce26992
	Removing intermediate container 6c845ce26992
	 ---> 0ab30541f7ea
	Step 4/5 : ADD service.yml /data
	 ---> a0344636b78c
	Step 5/5 : ENTRYPOINT ["top", "-b"]
	 ---> Running in 905a22119eb7
	Removing intermediate container 905a22119eb7
	 ---> f9ba0de68a00
	Successfully built f9ba0de68a00
	Successfully tagged test:v2.0
	```
	
	Dockerfile中每一个指令都是一个镜像层，上层镜像依赖下层镜像，只要某一层发生变化，其上层所有镜像层缓存均失效。
	
- **镜像调试**

	镜像在构建过程中也经常会失败，当出现失败时，我们可以进行调试，通过`docker run`命令可以运行失败指令的前一个指令成功构建的镜像层。
	
	```Dockerfile
	From centos
	ADD readme.txt /data
	EXPOSE 8080
	ADD service.yml /data/yaml/
	ENTRYPOINT ["top", "-b"]
	```
	
	```
	[root@vm ~]# docker build -t test:v2.0 -f Dockerfile docker
	Sending build context to Docker daemon  4.194kB
	Step 1/5 : From centos
	 ---> 9f38484d220f
	Step 2/5 : ADD readme.txt /data
	 ---> Using cache
	 ---> a097ddb31783
	Step 3/5 : EXPOSE 8080
	 ---> Using cache
	 ---> 0ab30541f7ea
	Step 4/5 : ADD service.yml /data/yaml/
	failed to copy files: lstat /data/docker/overlay2/026d474a8bae00c99e5b126df2ebb99128f6b2978eecb341db5cead0b89f2719/merged/data/yaml: not a directory
	```
	
	执行 `ADD service.yml /data/yaml`指令时发生错误，可以直接启动容器运行指令`EXPOSE 8080`构建的这一镜像层0ab30541f7ea。
	
	```bash
	[root@vm172-20-0-15 ~]# docker run -it 0ab30541f7ea sh
	sh-4.2# ls
	bin  data  dev  etc  home  lib  lib64  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
	sh-4.2# cd /data
	sh: cd: /data: Not a directory
	```
	
	进入之后，我们才恍能大悟，上面已经详细介绍过`ADD`指令了，当`ADD`指令的`dest`没有以`/`结尾时，Docker会把它当成是一个文件，在这里相当于把readme.txt的文件内容写到
	了`/data`这个文件中了，此时`/data`并不是文件夹，当执行`ADD service.yml /data/yaml/`命令时，Docker会自动去创建`/data`目录，此时已经有一个`/data`同名文件存在，
	所以创建失败。
	