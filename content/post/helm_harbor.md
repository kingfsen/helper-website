---
title: "Helm Harbor"
date: 2019-03-19T10:02:41+08:00
description: "开源镜像仓库服务Harbor集成Helm chart功能"
thumbnail: "img/helm.png"
Tags:
- helm
- harbor 
- chartmuseum
Categories:
- 容器生态
keywords:
- helm
- helm push
- harbor
- chartmuseum
---

### chartmuseum
chartmuseum是helm chart的仓库，它的存储层支持FileSystem以及各大云厂商的对象存储中间件，
默认支持阿里云的OSS、百度的BOS、Amazon S3、Microsoft Azure、Oracle、Openstack、Google等，
其他厂商需自行实现Storage的Backend接口方法，同时增加了缓存redis，提升系统处理效率，项目地址`https://github.com/helm/chartmuseum` 。

chartmuseum支持多租户的实现，它通过一个index-cache.yaml索引文件来维护租户上传记录，类似docker镜像中的manifest文件概念。

```Bash
--depth=0:
	Path: "/index.yaml"
	Repo: ""
--depth=1:
	Path: "/myrepo/index.yaml"
	Repo: "myrepo"
--depth=2:
	Path: "/myorg/myrepo/index.yaml"
	Repo: "myorg/myrepo"
--depth=3:
	Path: "/myorg/myteam/myrepo/index.yaml"
	Repo: "myorg/myteam/myrepo"
```

对于harbor来说，在harbor中项目名称是全局唯一，因此harbor集成helm的时候depth设置为1即可。index.yaml记录了当前repo中上传的chart清单，
chartmuseum在请求index相关接口时会维护清单内容是否与系统中真实文件列表的一致性，下面这个index-cache.yaml文件表示当前repo下有两个
charts分别为etcd、postgresql。

```yaml
apiVersion: v1
entries:
  etcd:
  - appVersion: 3.3.11
    created: "2019-03-13T09:58:43.260050243Z"
    description: etcd is a distributed key value store that provides a reliable way
      to store data across a cluster of machines
    digest: b98c70e5a384a3b5619c978fe3f6d9d386fcb09dea3b831a9802d7323ccff2b2
    engine: gotpl
    home: https://coreos.com/etcd/
    icon: https://bitnami.com/assets/stacks/etcd/img/etcd-stack-110x117.png
    keywords:
    - etcd
    - cluster
    - database
    - cache
    - key-value
    maintainers:
    - email: containers@bitnami.com
      name: Bitnami
    name: etcd
    sources:
    - https://github.com/bitnami/bitnami-docker-etcd
    urls:
    - charts/etcd-1.4.3.tgz
    version: 1.4.3
  postgresql:
  - appVersion: 9.6.2
    created: "2019-03-13T09:50:52.162006382Z"
    description: Object-relational database management system (ORDBMS) with an emphasis
      on extensibility and on standards-compliance.
    digest: bf9f5b811037706b1a928a94f6d1fa40cf73b2e3149660810dee929554bde3de
    engine: gotpl
    home: https://www.postgresql.org/
    icon: https://www.postgresql.org/media/img/about/press/elephant.png
    keywords:
    - postgresql
    - postgres
    - database
    - sql
    name: postgresql
    sources:
    - https://github.com/kubernetes/charts
    - https://github.com/docker-library/postgres
    urls:
    - charts/postgresql-0.19.0.tgz
    version: 0.19.0
generated: "2019-03-13T09:58:43Z"
```

chartmuseum对外提供的http api也就10个，支持BasicAuth、BeareAuth等鉴权认证方式以及https安全协议。

- GET /，欢迎页
- GET /health ， 健康检查
- GET /:repo/index.yaml，获取index.yaml文件
- GET /:repo/charts/:filename，获取指定的chart包，比如/kingfsen1026/charts/etcd-1.4.3.tgz
- GET /api/:repo/charts 获取repo下所有的charts包基本描述信息
- GET /api/:repo/charts/:name，获取repo下指定名字的chart包基本描述信息
- GET /api/:repo/charts/:name/:version，获取repo下指定名字和版本的chart包基本描述信息
- POST /api/:repo/charts，上传charts包
- POST /api/:repo/prov，上传chart包对应的prov文件
- DELETE /api/:repo/charts/:name/:version，删除repo下指定名字和版本的chart包

chartmuseum中有两个重要的参数就是AllowOverwrite、AllowForceOverwrite，会出现多次更新charts包的时候，务必将这两个参数设置为true。

### harbor

harbor从1.6.0开始已经支持chartmuseum功能了，harbor中把URL中带有chartrepo的请求都交给后面的chartmuseum服务处理，harbor中chartmuseum的服务endpoint
为`http://chartmuseum:9999/ `。对于普通的请求，harbor直接调用chartmuseum的http接口完成，对应文件上传harbor则是完全将请求转发给chartmuseum。

```Docker
4733260bc6bc        goharbor/nginx-photon:v1.7.0                "nginx -g 'daemon of…"   11 days ago         Up 11 days (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:4443->4443/tcp   nginx
8cf0b84192ba        goharbor/harbor-portal:v1.7.0               "nginx -g 'daemon of…"   11 days ago         Up 11 days (healthy)   80/tcp                                                             harbor-portal
3f6f8057af65        goharbor/harbor-jobservice:v1.7.0           "/harbor/start.sh"       11 days ago         Up 11 days                                                                                harbor-jobservice
75e5cc3d7465        goharbor/harbor-core:v1.7.0                 "/harbor/start.sh"       11 days ago         Up 11 days (healthy)                                                                      harbor-core
0f1693dfdcc7        goharbor/chartmuseum-photon:v0.7.1-v1.7.0   "/docker-entrypoint.…"   11 days ago         Up 11 days (healthy)   9999/tcp                                                           chartmuseum
5bd6d1c87735        goharbor/harbor-registryctl:v1.7.0          "/harbor/start.sh"       11 days ago         Up 11 days (healthy)                                                                      registryctl
5f4f28c15fb2        goharbor/redis-photon:v1.7.0                "docker-entrypoint.s…"   11 days ago         Up 11 days             6379/tcp                                                           redis
d56aec03b170        goharbor/harbor-db:v1.7.0                   "/entrypoint.sh post…"   11 days ago         Up 11 days (healthy)   5432/tcp                                                           harbor-db
ad5dc6d6eede        goharbor/registry-photon:v2.6.2-v1.7.0      "/entrypoint.sh /etc…"   11 days ago         Up 11 days (healthy)   5000/tcp                                                           registry
a29256d185c5        goharbor/harbor-adminserver:v1.7.0          "/harbor/start.sh"       11 days ago         Up 11 days (healthy)                                                                      harbor-adminserver
ce307d6dc619        goharbor/harbor-log:v1.7.0                  "/bin/sh -c /usr/loc…"   11 days ago         Up 11 days (healthy)   127.0.0.1:1514->10514/tcp                                          harbor-log
9a8acfda148b        goharbor/redis-photon:v1.7.0                "docker-entrypoint.s…"   2 weeks ago         Up 2 weeks             0.0.0.0:6379->6379/tcp                                             my-redis
```

### helm

helm包括两部分，Helm Client(Helm) 以及Helm Server(Tiller)。安装的方式很多种，可以直接下载二进制可执行程序安装。
进入 https://github.com/helm/helm/releases ，根据系统下载不同的版本，然后解压进行安装。

#### helm install

`Linux`

由于国内网络环境，不翻墙无法获取tiller镜像gcr.io/kubernetes-helm/tiller:v2.13.0，需事先下载好该镜像，然后tag之后存入本地的harbor仓库中。

```bash
docker tag gcr.io/kubernetes-helm/tiller:v2.13.0 hub.test.company.com/library/tiller:v2.13.0 
docker push hub.test.company.com/library/tiller:v2.13.0 
```

Tiller镜像与Helm安装包下载完成之后，执行命令安装

```bash
# tar -zxvf helm-v2.0.0-linux-amd64.tgz
# mv linux-amd64/helm /usr/local/bin/helm
# helm init --tiller-namespace kube-system --tiller-image=hub.test.company.com/library/tiller:v2.13.0
# kubectl get pods -n kube-system
# helm version
```

`Windows`

windows一般用于本地开发调试，其helm服务端tiller在本地运行，并非运行在kubernetes中，它通过配置连接到远程的kubernetes。
由于windows的cmd操作不便，这里以git bash进行操作，以本机为例。

- 解压安装包之后放在d:/tools/helm，该目录下有helm.exe以及tiller.exe
- 将kubernetes的服务$HOME下的.kube/config文件copy至本机的$HOME下
- 打开git bash，启动tiller

```bash
$ tiller
[main] 2019/03/19 14:18:21 Starting Tiller v2.13.0 (tls=false)
[main] 2019/03/19 14:18:21 GRPC listening on :44134
[main] 2019/03/19 14:18:21 Probes listening on :44135
[main] 2019/03/19 14:18:21 Storage driver is ConfigMap
[main] 2019/03/19 14:18:21 Max history per release is 0
```

从日志看tiller服务已经成功启动，端口44134，接着启动helm client。


- 创建系统环境变量HELM_HOST=localhost:44134，告诉helm client对应的tiller服务地址。
- 以管理员身份再启动一个bash窗口，执行helm init，如果未设置$HOME_HOST，helm init --host=localhost:44134

```bash
$ helm init
Creating C:\Users\SUNJINFU\.helm
Creating C:\Users\SUNJINFU\.helm\repository
Creating C:\Users\SUNJINFU\.helm\repository\cache
Creating C:\Users\SUNJINFU\.helm\repository\local
Creating C:\Users\SUNJINFU\.helm\plugins
Creating C:\Users\SUNJINFU\.helm\starters
Creating C:\Users\SUNJINFU\.helm\cache\archive
Creating C:\Users\SUNJINFU\.helm\repository\repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at C:\Users\SUNJINFU\.helm.
Warning: Tiller is already installed in the cluster.
(Use --client-only to suppress this message, or --upgrade to upgrade Tiller to the current version.)
Happy Helming!
```

helm init的时候默认添加`https://kubernetes-charts.storage.googleapis.com` 作为stable仓库，如果本地网络不通，则无法获取该仓库的
index.yaml文件，则helm初始化失败，虽然此时Helm在$HOME新建.helm目录成功，但因为网络问题没有新建repositories.yaml文件，通过执行命令
`helm repo list`发现不可用，为了解决这个问题，只需手动手动在$HOME/.helm/repository目录下把helm的repositories.yaml文件新建即可，添加如下内容：

```yaml
apiVersion: v1
generated: 2019-03-19T14:24:05.322548+08:00
repositories:
- caFile: ""
  cache: C:\Users\SUNJINFU\.helm\repository\cache\stable-index.yaml
  certFile: ""
  keyFile: ""
  name: stable
  password: ""
  url: https://kubernetes-charts.storage.googleapis.com
  username: ""
```

注意修改cache为本地对应的路径，文件新建完成之后，再次执行`helm repo list`检查helm命令是否可用。

如果只想使用helm客户端的部分功能，比如上传下载chart包，完全不需要tiller服务，解决办法是在helm初始化的时候带上flag参数，helm init –client-only。

```bash
$ helm version
Client: &version.Version{SemVer:"v2.13.0", GitCommit:"79d07943b03aea2b76c12644b4b54733bc5958d6", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.13.0", GitCommit:"79d07943b03aea2b76c12644b4b54733bc5958d6", GitTreeState:"clean"}
```

#### helm repo

现在我们的harbor服务`https://hub.test.company.com` 中的账户admin/Harbor12345有一个私有项目kingfsen1026，用户需要把charts包上传到这个项目下。

添加仓库

```bash
$ helm repo add hub https://hub.test.company.com/chartrepo/kingfsen1026 \
--username admin --password=Harbor12345
"hub" has been added to your repositories
```

路径中的chartrepo是固定的，kingfsen1026是项目名称，它是动态的。查看$HOME/.helm/repository/repositores.yaml

```yaml
apiVersion: v1
generated: 2019-03-19T14:24:05.322548+08:00
repositories:
- caFile: ""
  cache: C:\Users\SUNJINFU\.helm\repository\cache\stable-index.yaml
  certFile: ""
  keyFile: ""
  name: stable
  password: ""
  url: https://kubernetes-charts.storage.googleapis.com
  username: ""
- caFile: ""
  cache: C:\Users\SUNJINFU\.helm\repository\cache\local-index.yaml
  certFile: ""
  keyFile: ""
  name: local
  password: ""
  url: http://127.0.0.1:8879/charts
  username: ""
- caFile: ""
  cache: C:\Users\SUNJINFU\.helm\repository\cache\hub-index.yaml
  certFile: ""
  keyFile: ""
  name: hub
  password: Harbor12345
  url: https://hub.test.company.com/chartrepo/kingfsen1026
  username: admin
```

同时helm在$HOME/.helm/repository/cache会生成hub-index.yaml文件，该文件记录了用户项目kingfsen1026下全部的chart包描述信息。
当执行helm search xx的时候则直接搜索当前目录下所有index.yaml文件中的keywords关键字。
helm在请求url的时候会把username、password以Basic Auth塞在http请求头中，
harbor收到请求之后，进行授权校验，授权成功之后再把请求转给chartmuseum处理，harbor代理了chartmuseum的授权管理。

helm repo相关操作

> * helm repo add hub xx， 添加repository
> * helm repo remove hub， 删除repository
> * helm repo update， 更新所有repo的index.yaml
> * helm repo list，列出所有repository
> * helm repo index，在包含chart package的目录下生成index.yaml


#### helm fetch

拉取chart

```bash
$helm fetch hub/etcd
```

拉取指定版本chart

```bash
$helm fetch hub/etcd --version 1.4.3
```
最终下载的文件为etcd-1.4.3.tgz，如果事先没有执行helm repo add hub操作，可以以完整路径拉取chart包。

```bash
helm fetch https://hub.test.company.com/chartrepo/kingfsen1026/charts/etcd-1.4.3.tgz \
--username admin --password Harbor12345
```

#### helm push

helm push并不在helm的项目，官方以plugin的形式提供helm push，插件的安装流程阅读文章 [Helm插件安装原理](/post/helm_plugin_install)，
helm-push官方项目地址`https://github.com/chartmuseum/helm-push`。

helm plugin相关命令

> * helm plugin install xx， 安装插件
> * helm plugin remove xx，卸载插件
> * helm plugin list，列举插件
> * helm plugin update xx，更新插件

安装helm push

```bash
$helm plugin install https://github.com/chartmuseum/helm-push
Downloading and installing helm-push v0.7.1 ...
https://github.com/chartmuseum/helm-push/releases/download/v0.7.1/helm-push_0.7.1_windows_amd64.tar.gz
Installed plugin: push
```

安装成功之后，通过helm help即可看到helm push命令，同时在$HOME/.helm/plugins目录下即可看到插件对应的目录。helm push这个插件
是官方提供的，功能比较丰富。它支持上传.tgz以及文件夹方式上传，它首先会校验tgz文件或者文件夹中的Chart.yaml以及values.yaml是否合法等，
校验成功之后重新生成一个tgz文件上传到chartmuseum。 

推送etcd-1.4.3.tgz到hub

```bash
$ helm push etcd-1.4.3.tgz hub
Pushing etcd-1.4.3.tgz to hub...
Done.
```

helm push首先会解压该文件，然后抽取其中的文件，验证是否满足chart的基本结构，
然后重新按照标准在临时目录生成一个全新的name-version.tgz的包，推送成功之后删除临时目录。

推送文件夹

```bash
$helm push ./etcd hub
```

#### curl upload

helm push插件不安装也没关系，可以直接使用curl上传charts，后端harbor服务接口支持Content-Type=application/octet-stream
或者Content-Type=multipart/form-data，只上传单文件时哪种方式都行。

二进制上传

```bash
curl -v --data-binary "@postgresql-0.19.0.tgz" \
https://hub.test.company.com/api/chartrepo/kingfsen100/charts \
--header 'authorization: Basic ZjFUWldOeVpYUmW4nhjbmt0VFdsaGJ6VUhka0xVW3bz0=' \
--header 'content-type: application/octet-stream'
```

form-data上传

```bash
curl -v -F "chart=@postgresql-0.19.0.tgz" https://hub.test.company.com/api/chartrepo/kingfsen100/charts \
--header 'authorization: Basic ZjFUWldOeVpYUmW4nhjbmt0VFdsaGJ6VUhka0xVW3bz0='
```

如果同时上传chart、prov文件，务必使用form-data方式

```bash
curl -v -F "chart=@postgresql-0.19.0.tgz" -F "prov=@"postgresql-0.19.0.tgz.prov" \
https://hub.test.company.com/api/chartrepo/kingfsen100/charts \
--header 'authorization: Basic ZjFUWldOeVpYUmW4nhjbmt0VFdsaGJ6VUhka0xVW3bz0='
```