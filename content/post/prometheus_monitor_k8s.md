---
title: "Prometheus监控Kubernetes服务(一)"
thumbnail: "img/prometheus.png"
date: 2019-03-13T10:29:13+08:00
description: "Kubernetes中如何部署Prometheus监控服务"
Tags:
- prometheus
- kubernetes
Categories:
- 容器生态
---

### Prometheus功能结构
Prometheus是基于golang编写的一个开源监控项目，当前应用非常广泛，尤其是与当前的容器调度平台kubernetes融合，使用Prometheus之前，我们应该了解下它的使用场景，它不能用来进行大量无规则数据的收集，不能替代日志收集工具，同时由于Prometheus基于时间序列的sample采样收集，如果某个系统对自身的每个请求都要求精准监控，那Prometheus也不适合。Prometheus更多的用于基于时间序列的一个sample采样，统计分析，比如某个功能接口某段时间的调用频率、调用次数等。
![prometheus](/blog/prometheus_monitor_k8s/001.png)
橙色框 Pushgateway、Prometheus server、Prometheus web UI、Alertmanager都是prometheus自身提供的核功能组件，蓝色框是prometheus支持的服务自动发现方式，其他模块的实现都是第三方项目。对于prometheus server来说，所有的metrics数据都是通过Job定时轮询抽取过来的。prometheus也已经想到，某些数据源系统可能无规则或者生命周期短，更可能的是它们与prometheus server所在的网络不通，所以它又提供了一个Pushgateway组件，数据源系统可以通过http请求把metrics数据推送到Pushgateway，然后Prometheus server把Pushgateway的地址作为稳定的数据抽取源。

Prometheus的服务自动发现，当前有两种方式，支持原生的Kubernetes以及基于静态配置文件file_sd。Prometheus定时获取解析file_sd中配置的数据抓取源URL，因此可以借助一些第三方组件比如DNS、Consul、Docker Swarm、Scaleway等当发现新服务时，自动更新file_sd。

Prometheus数据的存储，默认是基于磁盘的一个tsdb时序数据库，当然它也支持远程数据读写，必须通过Adapter进行适配。
![prometheus](/blog/prometheus_monitor_k8s/002.png)
![prometheus](/blog/prometheus_monitor_k8s/003.png)

Prometheusd的Alertmanager、ui基本都被Grafana的组件替代。

### Prometheus服务部署

当前很多项目已经基于流行的容器调度平台Kubernetes进行部署，因此把Prometheus部署在Kubernetes中，可以直接让Prometheus基于rbac方式拉取Kubernetes以及Kubernetes中业务应用的各项数据指标。为方便管理，在Kubernetes中新建一个命名空间kube-prometheus，用来进行部署相关的部署服务。每个yaml文件编写之后，依次执行`kubectl apply -f xx.yml`。

```
apiVersion: v1
kind: Namespace
metadata:
   name: kube-prometheus
```
然后创建prometheus的RBAC涉及的资源，ClusterRole、ServiceAccount、ClusterRoleBinding。

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: kube-prometheus
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: kube-prometheus
```
创建Prometheus的配置，在Kubernetes中以ConfigMap形式创建资源，然后执行部署Prometheus Deployment的时候Kubernetes自动将配置文件挂载到对应的容器目录，Prometheus的详细配置后面再做简单介绍。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: kube-prometheus
data:
  prometheus.yml: |-
      global:
        scrape_interval:     15s
        evaluation_interval: 15s
      scrape_configs:
 
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
 
      - job_name: 'kubernetes-nodes'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics
 
      - job_name: 'kubernetes-cadvisor'
        kubernetes_sd_configs:
        - role: node
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
 
      - job_name: 'kubernetes-service'
        kubernetes_sd_configs:
        - role: node
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - action: labelmap
          regex: __meta_kubernetes_service_annotation_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_service_name]
          target_label: service_name
 
      - job_name: 'kubernetes-endpoints'
        kubernetes_sd_configs:
        - role: endpoints
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_host_ip]
          target_label: host_ip
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod_name
        - source_labels: [__meta_kubernetes_service_name]
          target_label: service_name
        - source_labels: [__meta_kubernetes_endpoint_ready]
          target_label: ready
 
      - job_name: 'kubernetes-java-endpoints'
        kubernetes_sd_configs:
        - role: endpoints
        metrics_path: /prometheus
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: prometheus
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod_name
        - source_labels: [__meta_kubernetes_pod_ready]
          target_label: ready
        - source_labels: [__meta_kubernetes_service_name]
          target_label: service_name
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_pod_host_ip]
          target_label: host_ip
```
创建Prometheus的deployment、service资源文件

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  labels:
    name: prometheus-deployment
  name: prometheus
  namespace: kube-prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - image: prom/prometheus:v2.3.2
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/etc/prometheus/prometheus.yml"
        - "--storage.tsdb.path=/prometheus"
        - "--storage.tsdb.retention=48h"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus"
          name: data
        - mountPath: "/etc/prometheus"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 500m
            memory: 892Mi
      serviceAccountName: prometheus
      nodeSelector:
        kubernetes.io/hostname: 172.10.3.15
      volumes:
      - name: data
        hostPath:
          path: /data/prometheus_tsdb
      - name: config-volume
        configMap:
          name: prometheus-config
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: prometheus
  name: prometheus
  namespace: kube-prometheus
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30003
  selector:
    app: prometheus
```

上面这个文件执行之后，Prometheus的服务则部署完成，上面我们通过端口映射已经将Prometheus的服务端口映射到宿主机的30003，浏览器直接访问:http://ip:30003。

![prometheus管理端](/blog/prometheus_monitor_k8s/004.png)

由于Prometheus的图形效果远逊于Grafana，接着在Kubernetes中部署Grafana服务。

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: grafana-core
  namespace: kube-prometheus
  labels:
    app: grafana
    component: core
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: grafana
        component: core
    spec:
      containers:
      - image: grafana/grafana:5.2.0
        name: grafana-core
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            cpu: 300m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 100Mi
        env:
          - name: GF_AUTH_BASIC_ENABLED
            value: "true"
          - name: GF_AUTH_ANONYMOUS_ENABLED
            value: "false"
          - name: GF_SECURITY_ADMIN_PASSWORD
            value: "CONSOLE123456"
        readinessProbe:
          httpGet:
            path: /login
            port: 3000
        volumeMounts:
        - name: grafana-data
          mountPath: /var/lib/grafana
      nodeSelector:
        kubernetes.io/hostname: 172.10.3.10
      volumes:
      - name: grafana-data
        hostPath:
          path: /var/lib/grafana
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: kube-prometheus
  labels:
    app: grafana
    component: core
  annotations:
    prometheus.io/scrape: 'true'
spec:
  type: NodePort
  ports:
    - port: 3000
      nodePort: 30002
  selector:
    app: grafana
    component: core
```

为了防止在Grafana服务重启时丢失上一次的创建好的配置信息，务必通过文件目录映射，持久化Grafana配置信息，Grafana的web端用户名默认admin，密码则是上面以环境变量注入的CONSOLE123456。通过浏览器访问 http://ip:30002/ ，输入用户名密码进行登录。
![grafana_index](/blog/prometheus_monitor_k8s/005.png)

登录成功之后，开始配置数据源，将Prometheus的访问地址配置暴露给Grafana，然后Grafana可以从Prometheus请求数据，渲染页面。

![datasource](/blog/prometheus_monitor_k8s/006.png)

解释一下上面的URL地址http://prometheus:9090 , 9090是Prometheus的service的服务端口，prometheus是service的名字，由于Prometheus与Grafana服务部署于同一个命名空间kube-prometheus，直接通过service名字即可访问，如果不是同一个命名空间，则必须http://prometheus.kube-promethues.svc.cluster.local:9090 。配置URL之后，点击保存即可，然后去Grafana官网下载一个kubernetes的Template，导入到当前的Grafana中。

![template_grafana](/blog/prometheus_monitor_k8s/007.png)

### 细节问题

> grafana 5.1以上版本需要的是472用户权限, `chown -R 472:472 /var/lib/grafana`

> 如果prometheus的数据存放目录是hostPath形式，需要赋予权限, `chown -R 65534:65534 prometheus_tsdb`










