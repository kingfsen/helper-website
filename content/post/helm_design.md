---
title: "Helm架构与中心化部署解决方案"
date: 2019-06-02T13:05:32+08:00
description: "详细介绍Helm架构原理以及中心化部署Helm客户端"
thumbnail: "img/helm.png"
Tags:
- helm
- swift
- tiller
- grpc
Categories:
- 容器生态
---
随着Google容器调度平台Kubernetes的开源，很多社区与项目逐渐涌现，Helm就是其中之一。Helm可以理解成是Kubernetes的包管理工具，
类似rpm与CentOS、dpkg与Debian的关系。在Helm中，安装包被称为Chart，Helm定义了包的文件结构以及语法规范，同时提供了很多管理Chart包
的命令行工具，对于Chart包的存储，Helm也提供了chartmuseum项目来搭建Chart仓库，开源Harbor仓库也可以集成chartmuseum，提供一套企业级Chart包存储方案。

## 基础架构

与rpm、dpkg不同，Helm的宿主环境是Kubernetes，而Kubernetes又是跨宿主机节点的集群平台，这就决定了Helm的功能结构，一个客户端根本无法提供有状态的服务体系，
所以Helm提供了命令行工具Helm Client以及对应的服务端组件Tiller。Helm Client是安装在宿主机节点上的命令行工具，Tiller提供gRPC服务，它Pod、Service的资源形式
部署在Kubernetes集群中，在后面的介绍中Helm组件指的都是Helm Client。 

![tiller](/blog/go_base/tiller.jpg)

Helm客户端如何与Tiller通信呢，首先想到的应该是上图中的一种直观模式，Helm直接通过Proto协议请求Tiller Server，关于gRPC的介绍请参考-[gRPC服务介绍](/post/grpc) 。
在Kubernetes集群中，Helm客户端不可能直接采取这种简单粗暴的方式与Tiller通信，Tiller默认的服务端口44134，Helm在初始化安装Tiller的时候，虽然部署了Tiller服务的Service
资源，但这个Service并没有将任何端口映射到宿主机上，也就是说不可能直接在宿主机外请求到这个Tiller Service服务，那么客户端是如何将请求发给Tiller?

![tiller](/blog/go_base/tiller_k8s.jpg)

Helm客户端与Tiller通信，主要使用portforward端口转发。Kubernetes提供了`kubectl port-forward`命令，对于那些没有部署Service的Pod，在宿主机上通过该命令可以直接将本地端口流量
转发到Pod端口上，这对于调试Pod中容器服务非常方便。`kubectl port-forward`实际上是通过Kubernetes的api接口`POST /api/v1/namespaces/{namespace}/pods/{name}/portforward`实现，
Kubernetes的portforward功能依赖了`socat`、`nsenter`工具，节点未安装这两个命令工具，在请求portforward时会直接报错，如下kubelet相关portforward源码。

```go
containerPid := container.State.Pid
socatPath, lookupErr := exec.LookPath("socat")
if lookupErr != nil {
        return fmt.Errorf("unable to do port forwarding: socat not found.")
}

args := []string{"-t", fmt.Sprintf("%d", containerPid), "-n", socatPath, "-", fmt.Sprintf("TCP4:localhost:%d", port)}

nsenterPath, lookupErr := exec.LookPath("nsenter")
if lookupErr != nil {
        return fmt.Errorf("unable to do port forwarding: nsenter not found.")
}

```

kubectl port-forward使用方法

```go
# kubectl port-forward tiller-deploy-7ccf9c8f54-xrzrf :9855 -n kube-system
Forwarding from 127.0.0.1:14893 -> 9855
Forwarding from [::1]:14893 -> 9855
```

上面命令执行成功之后，直接请求本地的14893端口，则可以转发到Kubernetes中tiller-deploy-7ccf9c8f54-xrzrf这个Pod的9855端口上，端口转发与Pod对应的Service没有任何关系，
它通过nsenter进入到Pod中指定容器命名空间，执行socat，直接将流量转发到Pod中容器进程提供的端口上。

## 部署方案

Helm提供的这种CS模式，要求与Tiller交互的每个Kubernetes节点，都必须安装Helm客户端，否则无法使用Helm功能。在多集群管理模式中，我们希望部署一个统一的Helm Web服务来充当Helm
客户端的角色，能够与所有Kubernetes集群中的Tiller服务交互。举个例子，某个酒店房间在Elong、Ctrip、Qunar、Meituan等OTA渠道售卖，这些OTA渠道都提供了自己的APP，那问题来了，
酒店操作员需要分别登录到各OTA的APP上去操作资源，这种场景下操作繁杂效率低下，所以出现了很多PMS管理系统。

![pms](/blog/go_base/pms.png)

在企业多集群模式中，也需要这样一个统一的Helm客户端，在用户的集群中，不可能直接通过NodePort来暴露集群的Tiller服务，最简单的方法就是通过APIServer代理Tiller服务，但是APIServer
提供的都是HTTP REST服务，这个时候我们都会想到只要开发部署一个HTTP服务包装Tiller服务即可，然后通过APIServer代理我们这个HTTP服务即可，此时请求Tiller服务只需以HTTP REST请求
APIServer即可。我们想到的方案，先驱们早就实现并且已经在github上开源，就是`appscode/swift`项目。

swift的功能结构很简单，之前的版本(如0.8)，会在两个端口分别提供gRPC服务以及http服务，http服务接收外部进入的请求，再请求本地的gRPC Server服务，gRPC Server请求Tiller Service服务。

![swift1](/blog/go_base/swift1.png)

swift后面的版本(如0.11)只会在一个端口监听，一个端口连接多个协议复用。

![swift1](/blog/go_base/swift2.png)

在Kubernetes集群外通过APIServer请求swift在集群中的Service，首先需要将swift的服务暴露为集群服务。在Kubernetes集群中将Pod的Service暴露为集群Service，只需在Service资源定义
中增加一个Label：`kubernetes.io/cluster-service: "true"`。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: swift
    kubernetes.io/cluster-service: "true"
  name: swift
  namespace: kube-system
  ...
```

通过`kubectl cluster-info`命令查看暴露的集群Service资源。

```go
# kubectl cluster-info
Kubernetes master is running at https://172.20.1.17:6443
KubeDNS is running at https://172.20.1.17:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
swift is running at https://172.20.1.17:6443/api/v1/namespaces/kube-system/services/swift:http/proxy
```

swift提供的查询releases列表的接口是`/tiller/v2/releases/json?all=true`，通过Kubernetes的proxy api代理请求，curl测试一下即可，这里报错没关系，没数据的原因。

```bash
# curl -X GET https://172.20.1.17:6443/api/v1/namespaces/kube-system/services/swift:http/proxy/tiller/v2/releases/json?all=true \
--header "Authorization: Bearer $TOKEN" --cacert /tmp/ca.crt   
{"code":13,"message":"runtime error: invalid memory address or nil pointer dereference"}
```

在每个Kubernetes集群中都部署swift，同时将swift Service暴露为集群Service，这样可以通过http rest与各集群中的Tiller间接交互。

![swift_tiller](/blog/go_base/swift_tiller.png)