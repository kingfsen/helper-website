---
title: "Helm插件安装原理详解"
date: 2019-03-15T16:40:22+08:00
thumbnail: "img/helm.png"
description: "Helm插件安装详细分析"
Categories:
- 容器生态
Tags:
- Helm
---

Helm是Kubernetes集群的安装包管理工具，它与Kubernetes的关系类似于RPM与Centos。Helm提供了安装插件方式去扩展其核心功能，插件主要在客户端执行，并且存放在$HELM_HOME的plugins目录中。
Helm的插件安装源可以支持多种形式，Helm插件是以plugin.yaml文件来组织的，这个文件是必须的，plugin.yaml的模板如下。

```
name: "push"
version: "0.7.1"
usage: "Please see https://github.com/chartmuseum/helm-push for usage"
description: "Push chart package to ChartMuseum"
command: "$HELM_PLUGIN_DIR/bin/helmpush"
downloaders:
- command: "bin/helmpush"
  protocols:
  - "cm"
useTunnel: false
hooks:
  install: "cd $HELM_PLUGIN_DIR; scripts/install_plugin.sh"
  update: "cd $HELM_PLUGIN_DIR; scripts/install_plugin.sh"
```

helm如果通过VCS安装插件，可以通过指定版本version参数来拉取指定的tag、branch、commit等，安装插件命令

```
$ helm plugin install https://github.com/technosophos/helm-template
```

### 插件安装
下面分析helm plugin install命令执行详细流程。

Helm通过上面命令即可获取到source参数，接着按步骤分析source的安装源是哪一种类型。

- Local path，直接调用os.Stat(source)，文件存在则表示是Local path，如/data/helm-push
- URL资源，判断source是否包含tgz/tar.gz尾缀，同时以https/http为前缀，满足则表示URL资源，如 `https://github.com/chartmuseum/helm-push_0.7.1.tar.gz`
- VCS，既不是本地文件，也不是URL资源包，则直接判定为VCS仓库源，如 `https://github.com/chartmuseum/helm-push.git`

插件源类型确定之后，则开始构造插件资源的安装器Installer。

- LocalInstaller
这种方式最简单，插件目录已经在本地文件系统。比如插件目录在/data/kingfsen/helm-push，helm首先校验该目录下是否存在plugin.yaml文件，校验通过之后，helm调用系统创建一个文件软链接，
`$HELM_HOME/plugins/helm-push -> /data/kingfsen/helm-push` 。
- HTTPInstaller
首先检查$HELM_HOME/cache/plugins目录是否存在对应的文件，不存在则先通过http请求下载文件，然后解压在/HELM_HOME/cache/plugins目录下。
文件在本地之后，接下来逻辑则和LocalInstaller一致。
- VCSInstaller
拉取仓库代码，切换到对应的version，接下来逻辑与LocalInstaller一致。

安装器执行之后，加载该plugin。因为Helm通过installer已经创建了软连接，因此可以加载$HELM_HOME/plugin/helm-push目录下中的plugin.yaml文件，然后将该yaml文件解析成对应的plugin元数据Metadata。
开始执行plugin中的Hook，plugin中的hook有三种类型，分别为`install`，`delete`， `update`，此时helm会选择install类型的命令进行执行，在执行hook命令之前，helm会把相关的参数注入到环境变量中。执行方式为sh -c $hook， hook则是plugin中对应的hook数据。

`sh -c cd $HELM_PLUGIN_DIR; scripts/install_plugin.sh`

某种意义上来说，hook中的提供的install命令更像是helm提供给第三方插件的一个初始化逻辑。Helm插件的命令不能与helm自身的命令名称重复，即plugin.yaml中的name属性是全局唯一。根据上面的流程分析，完全可以手动给Helm创建一个插件。

### 插件加载
Helm在执行命令时，会加载所有的一级命令以及所有已安装的插件命令。

> * Helm加载$HOME_HOME/plugins/目录下所有子目录的plugin.yaml，每一个插件解析为plugin Metadata元数据
> * 给插件命令提供一些相关的参数，如kube等
> * 解析出插件的命令以及命令对应的参数，即plugin.yaml中提供的command，参数以空格分开跟在命令后面
> * 通过Google提供的命令行库Cobra将插件命令构建为命令行程序




