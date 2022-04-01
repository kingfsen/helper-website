---
title: "Prometheus监控Kubernetes服务(二)"
date: 2019-03-15T09:48:52+08:00
thumbnail: "img/prometheus.png"
description: "Prometheus监控服务详解"
Tags:
- prometheus
- kubernetes
Categories:
- 容器生态
---

prometheus相关的服务已经在kubernetes中部署完成，请参阅文章 [Prometheus监控Kubernetes服务(一)](/post/prometheus_monitor_k8s/) 。

### Prometheus Label
Label在prometheus服务抓取中非常重要，通过标签重写或者标签过滤抓取目标等是非常强大的功能。除了目标本身自定义的标签，prometheus还会内置给每个目标一些基础标签，
比如job、instance、metrics、schema等。prometheus获取的初始标签一般以`__meta__`开头，我们可以根据需要针对部分标签进行自定义命名重写，这样prometheus就会为目标保存这些自定义标签，那些以__开头的标签则都会被丢弃。

> * [ source_labels: '[' <labelname> [, ...] ']' ] #它的值范围是目标原始标签列表中的任何标签组合
> * [ separator: <string> | default = ; ] #一般用来分隔多个labels,多半配合用在regex
> * [ target_label: <labelname> ] #重写之后的标签名
> * [ regex: <regex> | default = (.*) ] #用来匹配标签
> * [ replacement: <string> | default = $1 ] #结合regex使用，是regex匹配内容的子串内容
> * [ action: <relabel_action> | default = replace ] #结合regex使用，匹配到需要的标签之后，进行的操作

relabel_action的值范围列表如下

> * replace: label_source匹配regex之后，根据replacement规则提取label_source中对应字符串，设置为target_label的值
> * keep: 丢弃不匹配regex的原始标签
> * drop: 丢弃匹配regex的原始标签
> * labelmap: 提取匹配了regex的所有标签中的replacement
> * labeldrop: 移除任何匹配regex的标签
> * labelkeep: 移除任何不匹配regex的标签

在kubernetes中部署了一个简单的Java Springboot应用，系统暴露的merics接口不再是/merics，而是/prometheus，由于需要让prometheus取获取该应用在每个pod中的merics数据，所以job的角色必须是endpoint。
```
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
__meta_kubernetes_service_annotation等都是prometheus自动附加的前缀，每个目标能获取的原始标签都是prometheus内置功能，这个具体看官方文档。这个抓取任务kubernetes-java-endpoints的merics_path强制写成了/prometheus，它的默认值是/metrics。再通过keep动作过滤抓取的目标，由于kubernetes中的pod很多，不能让job无限制去抓取，
通过在pod的annotation中注入一个属性prometheus.io.scrape = true来控制，接着重写了5个标签。

- __meta_kubernetes_pod_name -> pod_name
- __meta_kubernetes_pod_ready -> ready
- __meta_kubernetes_service_name -> service_name
- __meta_kubernetes_namespace -> namespace
- __meta_kubernetes_pod_host_ip -> host_ip

访问prometheus后台，从 Status -> Target查看标签重写结果，从图中的endpoint看出，prometheus抓取merics数据的url是通过`__schema__ + __address__ + merics_path`构造出来，而`__address__`是prometheus内置的标签，所以最重要的就是构造出抓取merics数据的接口URL。

![relabel](/blog/prometheus_monitor_k8s_2/001.png)

接着看一下复杂点的探针服务的Label配置

```
  - job_name: 'kubernetes-pod-tcp-probe'
	kubernetes_sd_configs:
	- role: pod
	tls_config:
	  ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
	bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
	metrics_path: /probe
	params:
	  module: [tcp_connect]
	relabel_configs:
	- source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_tcp_probe]
	  action: keep
	  regex: true
	- source_labels: [__meta_kubernetes_pod_ip, __meta_kubernetes_pod_container_port_number]
	  action: replace
	  target_label: __param_target
	  regex: (.+);(.+)
	  replacement: $1:$2
	- target_label: __address__
	  replacement: blackbox-exporter.kube-prometheus.svc:9115
	- source_labels: [__param_target]
	  target_label: instance
	- action: labelmap
	  regex: __meta_kubernetes_pod_label_(.+)
	- source_labels: [__meta_kubernetes_namespace]
	  target_label: namespace
	- source_labels: [__meta_kubernetes_pod_name]
	  target_label: pod_name
```

首先第一条标签操作通过keep动作过滤出需要抓取的pod，第二条标签操作将_param_target标签值设置成 pod的ip:pod的端口，第三条操作直接将 `__address__` 的值设置成 blackbox-exporter.kube-prometheus.svc:9115 服务地址，由于`__address__`被重写了，第四条操作重新给instance附上当前pod的ip以及端口值，__param_xx开头的
label会直接以xx为名字给构造好的url作为参数，这样上述的抓取url即为http://blackbox-exporter.kube-prometheus.svc:9115/probe?moudle=tcp_connect&&target=podIp:port 。这样prometheus每次调用该url，探针服务器则以tcp方式去请求target，然后分析请求过程中的时间dns等信息回馈给prometheus。

![relabel](/blog/prometheus_monitor_k8s_2/002.png)

### 监控Java应用指标
#### Prometheus指标类型
prometheus的度量数据类型有四种，分别为Counter、Gauge、Histogram、Summary，具体的解释参见 https://prometheus.io/docs/concepts/metric_types/ 。
Counter类似计算器，只增不减，比较适用于记录请求次数、异常次数等指标。Guage类似仪表盘，可增可减，比较适用于记录cpu使用量、内存使用量、温度等指标。Histogram类似柱状图，用于展示数据分布情况。Summary计算在一定时间窗口范围内度量指标对象的总数以及所有对量指标值的总和。
springboot应用默认暴露度量数据采集接口/metrics，但是这个接口返回的数据不符合prometheus的规范，所以它无法处理。找一个springboot服务，访问一下/metrics接口，查看返回的部分数据。

`
{
    "mem":657928,
    "mem.free":251278,
    "processors":4,
    "instance.uptime":160001,
    "uptime":176230,
    "systemload.average":0.22900390625,
    "heap.committed":582144,
    "heap.init":126976,
    "heap.used":330865,
    "heap":1780736
}
`

springboot的metrics接口返回的是json格式数据，并且指标名都是以点号拼接，这对prometheus来说是无法处理的。prometheus对metric name的格式有严格的规定，必须匹配：[a-zA-Z_:][a-zA-Z0-9_:]*，
规范的数据格式是name{labelname="labelvalue"} value这样的text文本格式。


#### 开发步骤
其实上面这些指标都是应用系统级的一些指标，我们现在更关注的是应用业务指标，下面进入具体的开发步骤。

- 引入Jar包io.prometheus:simpleclient_spring_boot:0.1.0
- 编写指标收集工具类。

```Java
/**
 * some sample data :
 * container_console_request_total{request_name=GetCluster,status=success}
 * container_console_request_total{request_name=GetCluster,status=exception}
 * container_console_request_total{request_name=GetCluster,status=fail}
 * author:SUNJINFU
 * date:2018/9/14
 */
public class Monitor {
 
    private static final String STATUS_SUCCESS = "success";
 
    private static final String STATUS_FAIL = "fail";
 
    private static final String STATUS_EXCEPTION = "exception";
 
 
    private static final Summary RequestTimeSummary = Summary.build("container_console_request_milliseconds",
            "request cost time").labelNames("request_name", "status").register();
 
    private static final Counter RequestTotalCounter = Counter.build("container_console_request_total",
            "request total").labelNames("request_name", "status").register();
 
    public static void recordSuccess(String labelName, long milliseconds) {
        recordTimeCost(labelName, STATUS_SUCCESS, milliseconds);
    }
 
    public static void recordSuccess(String labelName) {
        recordTotal(labelName, STATUS_SUCCESS);
    }
 
    public static void recordException(String labelName, long milliseconds) {
        recordTimeCost(labelName, STATUS_EXCEPTION, milliseconds);
    }
 
    public static void recordException(String labelName) {
        recordTotal(labelName, STATUS_EXCEPTION);
    }
 
    public static void recordFail(String labelName, long milliseconds) {
        recordTimeCost(labelName, STATUS_FAIL, milliseconds);
    }
 
    public static void recordFail(String labelName) {
        recordTotal(labelName, STATUS_FAIL);
    }
 
    private static void recordTimeCost(String labelName, String status, long milliseconds) {
        if (StringUtils.isNotEmpty(labelName)) {
            RequestTimeSummary.labels(labelName.trim(), status).observe(milliseconds);
        }
    }
 
    private static void recordTotal(String labelName, String status) {
        if (StringUtils.isNotEmpty(labelName)) {
            RequestTotalCounter.labels(labelName.trim(), status).inc();
        }
    }
}
```

- 埋点，利用上面Monitor的静态方法，在程序执行重要点进行埋点。

```Java
private void doBusiness() {
    long t = System.currentTimeMillis();
    try {
        //business logic
		.......
        Monitor.recordSuccess("business_xx", System.currentTimeMillis() - t);
    } catch (Exception e) {
        Monitor.recordException("business_xx", System.currentTimeMillis() - t);
    }
    log.info("{} cost time:{}", action, System.currentTimeMillis() - t);
}
```
业务成功调用成功的recordSuccess方法，异常调用recordException方法记录，这种静态方法埋点的好处就是灵活，当然针对系统的http请求，可以编写一个拦截器，记录每一个http请求结果。
```Java
@Component
public class RequestTimingInterceptor extends HandlerInterceptorAdapter {
 
    private static final String REQ_PARAM_TIMING = "http_req_start_time";
 
    private static final Summary responseTimeInMs = Summary.build()
            .name("container_console_http_response_time_milliseconds")
            .labelNames("method", "handler", "status")
            .help("http request completed time in milliseconds")
            .register();
 
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response,
                             Object handler) {
        request.setAttribute(REQ_PARAM_TIMING, System.currentTimeMillis());
        return true;
    }
 
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response,
                                Object handler, Exception ex) {
        Long timingAttr = (Long) request.getAttribute(REQ_PARAM_TIMING);
        long completedTime = System.currentTimeMillis() - timingAttr;
        String handlerLabel = handler.toString();
        if (handler instanceof HandlerMethod) {
            Method method = ((HandlerMethod) handler).getMethod();
            handlerLabel = method.getDeclaringClass().getSimpleName() + "_" + method.getName();
        }
        responseTimeInMs.labels(request.getMethod(), handlerLabel,
                Integer.toString(response.getStatus())).observe(completedTime);
    }
}
```

然后把这个拦截器注册到springboot中

```Java
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "com.xx.xx")
public class WebConfiguration extends WebMvcConfigurerAdapter {
 
    @Autowired
    private RequestTimingInterceptor requestTimingInterceptor;   
 
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(requestTimingInterceptor);
    }
}
```
上面的指标都记录了请求的响应时间，在某个时间段范围内，可以大约计算出请求的平均响应时间。

- 暴露prometheus metrics

在springboot的启动类上添加注解@EnablePrometheusEndpoint，这样springboot应用就暴露了/prometheus接口，系统在启动的时候会打印出相关的日志，这个接口输出的数据符合prometheus规范。如果接口不能访问，请在springboot应用的启动配置中添加management.security.enabled=false。

- kubernetes部署

在service的yaml中增加annotation，prometheus根据该annotation属性抓取对应的所有endpoint。
```
kind: Service
apiVersion: v1
metadata:
  name: container-console-service
  annotations:
    prometheus.io.scrape: 'prometheus'
spec:
  type: NodePort
  selector:
    name: container-console
  ports:
  - protocol: TCP
    port: 80
    targetPort: *
    nodePort: *
```
查看发现的服务目标数量

![pod](/blog/prometheus_monitor_k8s_2/003.png)

