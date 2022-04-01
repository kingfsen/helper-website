---
title: "Docker镜像存储结构与原理"
date: 2019-04-25T14:21:03+08:00
thumbnail: "img/docker_driver.png"
description: "详细介绍Docker当前常用的storage sriver以及Docker如何管理和存储镜像文件"
keywords: "docker, storage driver, 镜像存储"
Tags:
- docker
- overlay
Categories:
- docker
---

Docker容器镜像存在哪儿，怎么存放? 

---

Docker容器在运行的过程中，只有小部分的数据可能需要写到容器可写层，因为大部分场景下，我们可以通过Docker volumes来写数据，
但是某些场景下，就是需要往容器可写层写数据，这就是Docker存储驱动Storage Driver出现的原因，这些存储驱动必须是采用层堆叠以及`copy-on-write`策略。

Docker采用可插拔的方式支持一些不同的存储驱动，存储驱动控制了镜像和容器在宿主机上如何管理以及存储。如果系统内核支持多种存储驱动，在没有明确设置
存储驱动时，Docker会根据优先级`(btrfs,zfs,overlay2,aufs,overlay,devicemapper,vfs)`顺序进行选择，排在前面的储存驱动并不总是会被Docker采用，
因为有些存储驱动对文件系统或者内核有硬性要求。

---

Docker支持如下storage driver:

- **overlay2** <br/>
	Docker优先采用的存储驱动，对于已支持该驱动的Linux发行版，不需要任何进行任何额外的配置。
	
- **aufs** <br/>
	Docker 18.06以下版本运行在Ubuntu 14.04以及kernel 3.13上，不支持`overlay2`，首选`aufs`。
	
- **devicemapper** <br/>
	默认的配置模式是`loopback-lvm`，但是在生产环境时Docker不推荐这种模式，它的性能非常差，我们需要将其配置成`direct-lvm`。
	对于早期的CentOS和RHEL，`devicemapper`是推荐的存储驱动，因为它们对应的内核版本不支持`overlay2`，不过，我们现在版本的CentOS与RHEL均已支持`overlay2`。
	
- **btrfs and zfs** <br/>
	Docker安装所在的宿主机文件系统是`btrfs`或者`zfs`，则Docker优先使用它们作为存储驱动，这些文件系统具有一些高级的选项配置，比如创建快照(snapshots)等。
	
- **vfs** <br/>
	Docker不推荐用于生产环境，只用于测试，性能比较差，可能无法支持copy-on-write策略。

---

> * 注意一些存储驱动要求使用特殊版本的文件系统，比如`aufs`仅在`Ubuntu`或者`Debian`系统被支持, 并且需要安装一些额外的依赖包，而`btrfs`仅在`SLES`被支持。

---

我们大多数人使用的应该都是Docker社区版本，下面列举了社区版Docker引擎在各Linux发行版中推荐的存储驱动。

| Linux发行版        | 推荐的存储驱动   |  备选存储驱动  |
| :--------   |:-----  | :----  |
| Ubuntu    | `overlay2`、<br/>`aufs`(运行在内核3.13上的Ubuntu 14.04) |`overlay`、`devicemapper`、<br/>`zfs`、`vfs`|
| Debian    | `overlay2`(Debian扩展版本)、<br/>`aufs`、`devicemapper`(老版本) |`overlay`、`vfs`|
| CentOS    | `overlay2` |`overlay`、`devicemapper`、<br/>`zfs`、`vfs`|
| Fedora    | `overlay2` |`overlay`、`devicemapper`、<br/>`zfs`、`vfs`|

> * `overlay` storage driver已经在Docker引擎(企业版)18.09中被废弃，它在未来的发行包中将被移除，Docker推荐使用`overlay`存储驱动的用户迁移到`overlay2`。
> * `devicemapper` storage driver已经在Docker引擎18.09版本中被废弃，它在未来的发行包中将被移除，Docker推荐使用`devicemapper`存储驱动的用户迁移到`overlay2`。

总之，Docker推荐使用的storage driver是`overlay2`，Docker安装时默认选择`overlay2`，对于Docker不推荐使用的storage driver，可以通过强制手动设置使用，可能会出现一些
意想不到的错误，上面介绍的这些都是基于Linux平台，不可能支持Windows或Mac平台。

---

Docker存储驱动需要文件系统结构的支持:

|存储驱动        | 支持的文件系统   |
| :--------   |:-----  |
| `overlay2`、 `overlay`    | `xfs`(with ftype=1)、`ext4` |
| `aufs`    | `xfs`、`ext4` |
| `devicemapper`    | `direct-lvm` |
| `btrfs`    | `btrfs` |
| `zfs`    | `zfs` |
| `vfs`    | any filesystem |


Docker各存储驱动的一些特点:

- `overlay2`、`aufs`、`overlay` 基于文件级操作，使用内存效率高，在大量写场景下容器可写层会变得非常大。
- `devicemapper`、`btrfs`、`zfs` 基于block级操作，在大量写场景下性能更好。
- 很少发生写操作同时镜像层数又很多的场景下，`overlay`表现的比`overlay2`更好，但是`overlay`会消耗更多的inodes，可能导致inode瓶颈。
- `btrfs`、`zfs`需要大量的内存支持。

---

查看当前Docker的存储驱动

```docker
[~]# docker info
Containers: 8
 Running: 8
 Paused: 0
 Stopped: 0
Images: 80
Server Version: 18.09.2
Storage Driver: overlay2
 Backing Filesystem: extfs
 Supports d_type: true
 Native Overlay Diff: false
Logging Driver: json-file
Cgroup Driver: cgroupfs
```

Docker的storage driver是`overlay2`，文件系统是`ext4`，可通过命令`cat /etc/fstab`查看有效的文件系统详细信息。
Docker以插拔式结构支持多种storage driver，但是各storage driver都使用镜像层叠以及copy-on-write策略来达到相同的效果，
下面具体介绍Docker推荐的`overlay2`工作原理，了解`overlay2`之后，便能理解Docker镜像以及容器的组织结构。

`OverlayFS`是一种比较先进的联合文件系统，比起`AUFS`，它的速度更快，实现起来相对更简单。Docker提供了两种实现，`overlay`是最初的版本实现，
`overlay2`是新版本实现，更稳定。`overlay2`要求内核4.0以上，或者RHEL、CentOS 3.10.0-514以上版本，通过命令`uname -a`查看。


---

Docker在运行一段时间之后，如果手动修改了storage driver，那本地系统中的Docker镜像以及容器均无法进入，无法查看，当storage driver再次切换回来时，
之前的镜像与容器可再次使用，Docker根目录下的文件系统，我们不应该进行任何手动操作，它应该被Docker自动管理。

修改Docker的storage driver

- 暂停Docker服务
	
	```bash
	systemctl stop docker
	```
- 备份Docker根目录(/var/lib/docker)

	```bash
	cp -au /var/lib/docker /var/lib/docker.bk
	```
- 编辑Docker配置文件/etc/docker/daemon.json，不存在则新建，增加storage driver配置。

	```bash
	{
	  "storage-driver": "overlay2"
	}
	```
- 启动Docker

	```bash
	systemctl start docker
	```
	
通过`docker info`命令查看Docker存储驱动是否已经变更。

---

**overlay2工作原理**

`OverlayFS`在单台Linux上将两个目录进行层叠处理，最终将他们显示为另外一个单独的目录视图，这些被处理的每个目录都被称为层，视图统一的过程则称为联合挂载。
多个目录进行层叠，肯定具有上下层关系，`OverlayFS`将下层的目录称为`lowerdir`，上层的目录称为`upperdir`，被暴露的统一视图目录称为`merged`。
`overlay2`支持`OverlayFS`中的下层数目最大为`128`，通过`docker build`或者`docker commit`命令构建镜像时勿设置过多层。

![docker_data](/blog/docker_data/002.png)


下面以拉取ubuntu:latest镜像为例，讲解其在本地存储结构。

```docker
docker pull ubuntu
Using default tag: latest
latest: Pulling from library/ubuntu
898c46f3b1a1: Pull complete 
63366dfa0a50: Pull complete 
041d4cd74a92: Pull complete 
6e1bee0f8701: Pull complete 
Digest: sha256:017eef0b616011647b269b5c65826e2e2ebddbe5d1f8c1e56b3599fb14fabec8
Status: Downloaded newer image for ubuntu:latest
```

ubuntu:latest镜像一共有`4`层，那么在Docker的层文件系统中则会生成`5`个文件夹。

```bash
drwx------ 3 root root  4096 Apr 26 15:04 9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b
drwx------ 4 root root  4096 Apr 26 15:04 c47ec19c6123f5a75d9afbe9d9a631ecaa3469ad670bbb2b1b9ce5e64e361c7d
drwx------ 4 root root  4096 Apr 26 15:04 e8381b82a136d9cbad47970923b3dec9d6abaaffe08268a326ccf114a4f46d06
drwx------ 4 root root  4096 Apr 26 15:04 4a00cadd4488f55ad2378a9d2d915dcff733244fc96217b89f827c5f8959b34b
drwx------ 2 root root 20480 Apr 26 15:04 l
```

这个`l`目录中都是对应镜像层id的短标识符，都是对镜像层内容目录`diff`的一个符号链接，作用就是防止在使用`mount`命令时参数过长，这取决于你的Docker根目录长度，
默认`/var/lib/docker`。

```bash
I7KQTMK5P2FJUBEVF5AG4YX7OI -> ../9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b/diff
6W32BOAFLZDOGIACBFAYPIMFWU -> ../c47ec19c6123f5a75d9afbe9d9a631ecaa3469ad670bbb2b1b9ce5e64e361c7d/diff
N2VPKSUXNXCWKUOSWR2HXFLWME -> ../e8381b82a136d9cbad47970923b3dec9d6abaaffe08268a326ccf114a4f46d06/diff
TFMVIO6TPMR4AGFI2O2TWAWZQ7 -> ../4a00cadd4488f55ad2378a9d2d915dcff733244fc96217b89f827c5f8959b34b/diff
```

每个镜像层目录中包含了一个文件`link`，文件内容则是当前层对应的短标识符，镜像层的内容则存放在`diff`目录，查看ubuntu:latest第一层，也就是最下层目录。

```bash
[root@vm ]#ls /var/lib/docker/overlay2/9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b
[root@vm 9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b]# ls
diff  link
[root@vm 9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b]# cat link
I7KQTMK5P2FJUBEVF5AG4YX7OI
[root@vm 9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b]# ls diff
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

从镜像最底层的往上每一层，都包含一个文件`lower`指向了其所有的父层，其文件内容则是父层id的短标识符。
其中`diff`目录包含了镜像层内容，`work`目录被`OverlayFS`内部使用。

```bash
[root@vm9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b]# cd ../c47ec19c6123f5a75d9afbe9d9a631ecaa3469ad670bbb2b1b9ce5e64e361c7d
[root@vm c47ec19c6123f5a75d9afbe9d9a631ecaa3469ad670bbb2b1b9ce5e64e361c7d]# ls
diff  link  lower  work
[root@vm c47ec19c6123f5a75d9afbe9d9a631ecaa3469ad670bbb2b1b9ce5e64e361c7d]# cat lower
l/I7KQTMK5P2FJUBEVF5AG4YX7OI
[root@vm c47ec19c6123f5a75d9afbe9d9a631ecaa3469ad670bbb2b1b9ce5e64e361c7d]# cd diff
[root@vm diff]# ls
etc  sbin  usr  var
```

最上层镜像4a00cadd4488f55ad2378a9d2d915dcff733244fc96217b89f827c5f8959b34b的`lower`文件内容

```bash
l/N2VPKSUXNXCWKUOSWR2HXFLWME:l/6W32BOAFLZDOGIACBFAYPIMFWU:l/I7KQTMK5P2FJUBEVF5AG4YX7OI
```

下面启动一个运行ubuntu:latest镜像的容器

```docker
docker run -it -d --name ubuntu ubuntu
```

容器启动之后，查看/var/lib/docker/overlay2目录，发现多了两个目录。

```bash
drwx------ 4 root root  4096 Apr 26 16:11 8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0-init
drwx------ 5 root root  4096 Apr 26 16:11 8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0
```

首先进入init目录，查看其`lower`文件内容，正是`4`个镜像层短标识符，但是这一层并不是容器层，而是Docker在启动容器时默认加的一个辅助层，
用于给容器配置host等运行时的相关信息。

```bash
[root@vm 8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0-init]# ls
diff  link  lower  work
[root@vm 8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0-init]# cat lower
l/TFMVIO6TPMR4AGFI2O2TWAWZQ7:l/N2VPKSUXNXCWKUOSWR2HXFLWME:l/6W32BOAFLZDOGIACBFAYPIMFWU:l/I7KQTMK5P2FJUBEVF5AG4YX7OI
```

再查看不带init尾缀的目录

```bash
[root@vm 8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0]# ls
diff  link  lower  merged  work
[root@vm 8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0]# cd diff
[root@vm diff]# cat ../lower
l/MGUMOBQVXCENNH3EYBVSA3FXIP:l/TFMVIO6TPMR4AGFI2O2TWAWZQ7:l/N2VPKSUXNXCWKUOSWR2HXFLWME:l/6W32BOAFLZDOGIACBFAYPIMFWU:l/I7KQTMK5P2FJUBEVF5AG4YX7OI
[root@vm diff]# cd ../merged
[root@vm merged]# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

这个目录比镜像层目录多了一个子目录`merged`，这个`merged`目录则是这些镜像层以及容器层挂载之后的统一视图目录，
也就是进入容器之后能看见的文件系统了。为了验证一下，我这里手动在当前`merged`目录新建一个youendless.txt文本(这里仅用于测试，请勿手动操作Docker存储目录)，
然后进入容器中查看是否存在。

```bash
[root@vm ~]# docker exec -it 7334cfbc206d sh
# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var youendless.txt
# pwd
/
```

在上面容器中通过命令`touch okk.txt`新增一个文本文件，然后查看8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0的diff目录，

```bash
[root@vm1 diff]# cd /var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/diff
[root@vm1 diff]# ls
okk.txt  youendless.txt
```

很显然，这个目录才是真正的容器层目录，到这里，想必已经深刻理解了Docker镜像以及容器组织结构关系了，
当然不同的Docker版本在处理方式有差别，有些版本可能在每一层包括容器层就生成一个`merged`目录，作为当前层与其父层的统一视图。

执行命令查看overlay联合挂载信息

```bash
[root@vm /]# mount | grep '8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/merged'
overlay on /var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/merged type overlay (rw,relatime,
lowerdir=/var/lib/docker/overlay2/l/MGUMOBQVXCENNH3EYBVSA3FXIP:/var/lib/docker/overlay2/l/TFMVIO6TPMR4AGFI2O2TWAWZQ7:/var/lib/docker/overlay2/l/N2VPKSUXNXCWKUOSWR2HXFLWME:/var/lib/docker/overlay2/l/6W32BOAFLZDOGIACBFAYPIMFWU:/var/lib/docker/overlay2/l/I7KQTMK5P2FJUBEVF5AG4YX7OI,
upperdir=/var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/diff,
workdir=/var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/work)
```

查看容器信息(忽略无关信息)

```docker
"GraphDriver": {
	"Data": {
		"LowerDir": "/var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0-init/diff:/var/lib/docker/overlay2/4a00cadd4488f55ad2378a9d2d915dcff733244fc96217b89f827c5f8959b34b/diff:/var/lib/docker/overlay2/e8381b82a136d9cbad47970923b3dec9d6abaaffe08268a326ccf114a4f46d06/diff:/var/lib/docker/overlay2/c47ec19c6123f5a75d9afbe9d9a631ecaa3469ad670bbb2b1b9ce5e64e361c7d/diff:/var/lib/docker/overlay2/9955b547c10c465488960ecba07dcb8ace596c972e3ad8d391afc20cb6aab46b/diff",
		"MergedDir": "/var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/merged",
		"UpperDir": "/var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/diff",
		"WorkDir": "/var/lib/docker/overlay2/8985decfb3a78124f061c6d5995a158ca7126c7ccdf4ab7d66853107fb55b4a0/work"
	}
```