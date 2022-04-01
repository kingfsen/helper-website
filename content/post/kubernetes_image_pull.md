---
title: "kubernetes镜像拉取失败解决方法"
date: 2019-03-29T10:17:45+08:00
thumbnail: "img/kubernetes.png"
description: "kubernetes启动Pod时镜像拉取失败解决方法详解"
Tags:
- kubernetes
Categories:
- 容器生态
---

Docker Hub以及利用开源harbor项目搭建的镜像仓库服务，对于Docker Client发起的docker login、docker push、docker pull等命令都会做基本的用户认证，
最简单常用的认证方式就是Basic Auth，即在发起的http请求头中添加一个Authorization，其值为base64(username:password)，当前Docker Client都是这么处理。

在Kubernetes中，Secret资源对象用来存储和管理一些敏感信息，比如密码、Auth Token以及SSH keys，把这些敏感信息放入Secret对象中，相对来说更安全更灵活。
Kubernetes可以通过环境变量、文件挂载等方式将Secret信息推到每一个Pod中，通过文件挂载形式还能使Secret在pod中实时更新，Kubernetes统一管理。

Kubernetes中调度Pod成功之后，则开始拉取指定镜像启动容器，在Deployment对象中有几个与镜像拉取相关的重要配置参数。

- spec.template.spec.containers[n].image，容器启动时的镜像
- spec.template.spec.imagePullSecrets，Secret中定义了镜像所在仓库的用户名密码
- spec.template.spec.containers[n].imagePullPolicy，定义了镜像拉取策略

imagePullPolicy决定了是否发起镜像下拉请求，它的值范围Always、Never、IfNotPresent，默认为IfNotPresent，但标签为:latest的镜像默认为Always。

- Always，不管宿主机上镜像是否存在，都会发起一次下拉镜像请求
- Never，不管宿主机上镜像是否存在，都不会发起下拉镜像请求
- IfNotPresent，如果宿主机上镜像不存在，则向仓库发起下拉镜像请求

在Kubernetes中执行应用部署命令之后，通过命令`kubectl get pods`查看pod状态时，经常会遇见Pod的状态是ErrImagePull或者ImagePullBackOff，出现这种情况就一步一步分析。

```
kube-system       monitor-6c7fdcd477-jvqjc                 0/1       ImagePullBackOff   0          1h
```

1. 执行命令 

	```
	kubectl describe pod monitor-6c7fdcd477-jvqjc -n kube-system
	```
	
	或者```kubectl get events```，查看是否能发现具体的错误信息，由于kubernetes中错误信息不是很明显，通常只会展示error。
2. 在宿主机上执行docker login 检查用户名密码是否正确，接着执行docker pull imageId，操作均成功，表示镜像仓库服务正常。
3. 由于kubernetes中的pod网络可能与宿主机网络不一致，进入某个kubernetes的pod中，可以通过部署curl的pod用于网络测试，检查镜像仓库地址是否预期一致，不一致请把镜像仓库正确的域名或者host配置在集群dns中。
4. 查看应用部署中的deployment对应yaml中的imagePullPolicy，如果机器上无镜像，同时imagePullPolicy为Never，则镜像无法拉取。
5. 查看deployment对应yaml中的imagePullSecrets，其中的name就是secret的名字，如果拉取的是私有镜像，imagePullSecrets是必须的，没有secret，拉取镜像时请求
	仓库的http请求头Authorization则为空，仓库授权校验肯定不通过直接返回401错误，而kubernetes则可能直接显示error。
	注意secret是区分namespace的，容器启动时都是使用当前容器所在pod的namespace中的secret，执行命令检查secret是否存在。
	
	```
	kubectl get secret xxx -n kube-system -o yaml
	```
6. 当前namespace下对应的secret也存在，那就继续检查secret中的信息，取出上一步执行结果中显示的dockercfg字段对应的value值，应该是一长串base64编码的字符串，
	类似eyJodWIua2NlLmtzeXVeikkskseYELSH8sse，解码看一下具体的信息。
	
	```
	echo 'eyJodWIua2NlLmtzeXVeikkskseYELSH8sse' | base64 --decode
	```
	
	解码之后数据(mock数据，并非真实数据)
	
	```JSON
	{"hub.test.company.com":{"username":"98766743","password":"somebaby","email":"localhost","auth":"xxeieESrweSXs="}}
	```
	
	检查用户名、密码是否正确，用户名密码正确，还要查看域名是否与image包含的域名一致，如果deployment对应的yaml中的image为hub.test2.company.com/nginx/nginx:1.12.1，
	如果这样，即使用户名、密码正确，kubernetes也不会将Authorization放在请求仓库的http header中，也会导致镜像下载失败。
7. 发现secret中信息不对，则可以将该secret删除，然后重新创建，假设deployment中对应的imagePullSecrets中的name为hub.test.company.com.key。
	删除secret
	
	```
	kubectl delete secret hub.test.company.com.key -n kube-system
	```
	
	新建secret
	
	```Bash
	kubectl create secret docker-registry hub.test.company.com.key -n kube-system --docker-server=hub.test.company.com  
	--docker-username=2380997 --docker-password=RABC123456 --docker-email=test@company.com
	```
	
	注意创建时没指定namespace，那么默认为default。secret创建成功之后，重新部署即可。