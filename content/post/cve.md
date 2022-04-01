---
title: "Docker镜像扫描之CVE漏洞数据源"
date: 2019-04-30T16:32:25+08:00
description: "详细介绍如何从各开源社区获取详细的CVE数据，然后搭建自己的CVE数据库"
Tags:
- Clair
categories:
- Docker
---

[CVE](https://cve.mitre.org/cve/search_cve_list.html)，全称Common Vulnerabilities and Exposures，指的是公共已知的网络漏洞缺陷，每个漏洞都会分配一个唯一的标志CVE ID(如CVE-2008-7220)，
漏洞的相关信息由CNA维护，各系统厂商暴露出的漏洞通过CVE-ID均可在[CVE](https://cve.mitre.org/cve/search_cve_list.html) 社区跟踪查询。

既然要搭建自己的CVE数据库，我们需要知道的是在某个Linux系统版本，某个软件包产生了哪些漏洞，又在什么版本修复了哪些漏洞，
这些详细数据需要到各系统厂商维护的社区去获取，目前主要有5个开放社区，分别是alpine、debian、oracle、rhel以及ubuntu，各社区通过其官网
或者github持续维护软件包在其系统版本上的产生的各种漏洞数据，不过它们各自提供的数据格式差异很大，我们获取的时候需要进行适配。

随着Google的容器调度平台Kubernetes开源问世，[CoreOS](https://coreos.com/)也开始备受关注，开源分布式KV数据存储系统etcd，网络组件项目flannel等都是其产物。
我们现在要说的是另外一个项目[Clair](https://github.com/coreos/clair) ，该项目用于支撑Docker镜像扫描功能，它定时从各社区同步CVE数据，然后解析汇总保存在自己的
PostgreSQL数据库中，当接收到镜像扫描请求时，直接将镜像中的软件版本数据与数据库比对一番，即可输出当前镜像软件包已存在的各级别漏洞。

在Clair中，漏洞等级有以下几种:

- `Unknown` 社区暂未给出优先级或者Clair当前未同步支持
- `Negligible` 几乎无任何影响的漏洞，不会单独给出一个针对性的修复更新版本
- `Low` 低风险等级，一般会在更高风险等级漏洞修复版本中修复
- `Medium` 中风险等级，如跨域、用户权限暴露等，会有针对性版本修复
- `High` 高风险等级，漏洞在大多数人默认的安装版本中出现，如服务出错、数据丢失、root权限恶意获取等
- `Critical` 最高风险等级，几乎所有版本都存在某漏洞，如造成用户大量数据丢失

Clair通过参数控制同步漏洞数据周期，设置为0则表示不同步，默认12h。

```yaml
clair:
  updater:
    interval: 12h
```

Clair使用数据库表记录了上一次在各开源社区同步CVE数据的状态点，比如key为alpine-secdbUpdater的数据记录的是上一次同步alpine漏洞状态，
value为9d09b2a1e6e04e419e2488d2914bd6f383f62b76指的是git commit，其中的updater/last记录的是系统上一次CVE全部同步完成时间，Clair
结合系统参数interval来控制同步CVE漏洞数据的执行时间。

![keyvalue](/blog/cve/001.png)

从各开源社区获取CVE数据之后，Clair会进行数据解析，将各社区CVE数据统一结构，每一个漏洞都是一个database.Vulnerability对象。

```go
v := &database.Vulnerability{
  Severity: database.UnknownSeverity,
  Name: "CVE-2018-19044",
  Link: "https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-19044",
  FixedIn: []database.FeatureVersion{
    {
      Feature: database.Feature{
        Namespace: database.Namespace{
          Name:          "alpine:v3.9"
          VersionFormat: "pkg",
        },
        Name: "keepalived",
      },
      Version: "2.0.11-r0",
    },
  },
}
```

Severity为漏洞风险等级，Vuluerability.Name为CVE-ID，Link为CVE详情链接，Namespace.Name为操作系统，VersionFormat软件管理包，
如centos为rpm，alpine为pkg，Feature.Name为软件包名字，Version为软件包版本。

下面详细介绍Clair同步各社区CVE数据的策略

- alpine
  
    alpine维护的CVE数据源项目在github上，项目地址：https://github.com/alpinelinux/alpine-secdb 。
  
    ![keyvalue](/blog/cve/002.png)
    
    Clair首先通过查看当前目录是否存在alpine-secdb目录，存在则执行git pull命令拉取更新，否则执行git clone命令克隆该项目到本地。
    项目更新完成之后，开始判断是否需要进行数据解析，执行命令`git rev-parse HEAD`获取最新的commit，将该commit与数据库中记录的上一次
    commit对比，如果一致表示当前项目并无新的数据提交，无需同步，否则开始下面的数据解析流程。
    
    1. 解析namespace，Clair中namespace的概念就是操作系统版本的意思，如alphine:v3.3，centos:6.0。
    2. 解析CVE详细数据，v3.3 ~ v3.9指的是alpine的版本号，每个文件夹下有两个yaml文件，分别是main.yaml、community.yaml，两个文件中的数据格式是一致的，
    main.yaml指的是alpine自身相关组件产生的漏洞，而community则是社区一些安装包产生的漏洞，截取其中一段进行解释。
    
    ```yaml
    distroversion: v3.9
    reponame: community
    urlprefix: http://dl-cdn.alpinelinux.org/alpine
    apkurl: "{{urlprefix}}/{{distroversion}}/{{reponame}}/{{arch}}/{{pkg.name}}-{{pkg.ver}}.apk"
    packages:
    - pkg:
      name: keepalived
      secfixes:
        2.0.11-r0:
          - CVE-2018-19044
          - CVE-2018-19045
          - CVE-2018-19046
  - pkg:
      name: libraw
      secfixes:
        0.19.2-r0:
          - CVE-2018-20363
          - CVE-2018-20364
          - CVE-2018-20365
        0.18.6-r0:
          - CVE-2017-16910
    ```
    从文件中看出，在alphine:3.9操作系统上版本为`2.0.11-r0`的`keepalived`修复了CVE-2018-19044、CVE-2018-19045、CVE-2018-19046漏洞，然后将每一个CVE漏洞封装成一个
    Vulnerability。
    
- debian

    debian提供了一个http接口用于获取其CVE漏洞数据，接口直接返回application/json数据，
    URL: https://security-tracker.debian.org/tracker/data/json 。Clair利用sha1对http请求返回的body数据计算一个摘要，
    计算出的摘要和数据库中上一次的摘要一致，则不更新，否则解析json数据，接口返回数据样例如下。
    
    ![keyvalue](/blog/cve/003.png)
    
    debian各代码名字与数字版本的转换关系如下。
    
    ```bash
    squeeze: 6
    wheezy:  7
    jessie:  8
    stretch: 9
    buster:  10
    sid:     unstable
    ```
    
- rhel

    rhel数据源URL：https://www.redhat.com/security/data/oval/ 。
    Clair首先http请求URL，获取结果之后，逐行解析文本，筛选出符合匹配表达式`com.redhat.rhsa-(\d+).xml`的行，将匹配出的
    批次号与数据库中批次号比对，比数据库中批次号大则添加到待处理文件列表。首次解析rhel的CVE数据非常耗时，因为要解析处理所有
    的批次文件，之后则是增量更新，只处理新增的批次文件。
    
    ![keyvalue](/blog/cve/004.png)
    
    待处理文件列表不为空，则开始解析文件列表中文件数据，通过http请求获取xml文件内容。
  
- oracle

    oracle数据源URL: https://linux.oracle.com/oval/ 。
    oracle的CVE数据处理方式与rhel非常相似，不做过多详细介绍。
  
- ubuntu
    
    CVE数据维护项目git URL: https://git.launchpad.net/ubuntu-cve-tracker 。
    
Clair最后会将各社区的CVE数据再汇总组合，存入PostgreSQL中。

拥有各开源社区CVE数据源之后，则可以搭建自己的CVE数据库，继而搭建个人软件扫描服务。
  