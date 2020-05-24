---
title: K8S自定义指标弹性伸缩
date: 2020-05-25 00:32:18
toc: true
categories: [工作, 程序员, k8s]
tags: [k8s, docker]

---


# K8S自定义指标弹性伸缩

| 作者 | 版本 | 日期 | 审核人 | 备注 |
| ---- | :--: | :------: | :----: | :--: |
| 危奇 | 1.0 | 20191023 | 黄歆 | 初稿 |

## 背景

  一般业务都会存在忙/闲时间段, 如斗鱼相关业务, 大部分对外接口服务压力集中在晚上; 又比如某大型赛事开始前, 相关服务压力会剧增; 又比如某大主播直播间出现精彩时刻, 相关弹幕服务流量激增; 可以总结, 某些场景下, 相关服务压力是可以提前预见的, 可提前配置实例扩容, 而更复杂场景下, 无法提前预见, 需要基础设置根据相关监控指标, 动态地扩容相关核心服务, 避免出现服务无法响应的悲剧.

  K8S针对无状态服务(Deployment)提供了很多DevOps标准解决方案, 如滚动升级, 健康检查, 亲和性策略,弹性伸缩等等, 其中`弹性伸缩`是VM时代的标配功能, K8S也将其作为一个重要功能在版本更新中不断将其完善, 从一开始只支持CPU指标的弹性伸缩, 到近期版本(1.7)以后, 支持各类数据源指标的弹性伸缩.

  斗鱼容器平台早期只支持手动扩缩容, 随着K8S版本的迭代, 后续支持CPU 和内存指标的扩缩容, 但`相关开发者反应, 基于CPU和内存的指标无法直接映射到服务流量指标上`, 导致扩容很慢, 影响了核心服务质量; 一开始解决办法准备根据监控系统提供的指标做到各类自定义指标的扩缩容(如QPS), 但根据预研相关资料, 发现最新的K8S-HPA已经能根据Prometheus提供的指标弹性扩缩容, 因此决定不重复"造轮", 直接用K8S原生提供的弹性伸缩功能(HPA). 



## 原理

![](https://cdn.jsdelivr.net/gh/matrixcall/bed001@master/img/2019/01/20191216144934.jpg) 





  如图1所示, 在K8S新架构中(1.7后), 将相关指标通过接口的形式规范化, 即通过Metrics Aggegator 组件统一对外提供所有指标查询服务, 原生Metrics Server实现了CPU, 内存等相关指标查询接口, Prometheus-adapter提供了以普罗米修斯为数据源的其它指标查询接口; Metrics Aggegator 提供了类似Nginx转发的功能, 如果查询路径是 /apis/metrics.k8s.io, 则查询的是Metrics Server; 如果查询路径是 /apis/custom.metrics.k8s.io , 则查询的是自定义指标接口.


![](https://cdn.jsdelivr.net/gh/matrixcall/bed001@master/img/2019/01/20191216145018.png)


图2 展示的是图一部分自定义指标的细节说明, 核心组件是Kube-prometheus-adapter, 简单的说来, 它做了两件事: 

- 将K8S的查询语句翻译为Prometheus的查询语句;
- 将Prometheus查询出来的结果过滤+翻译成K8S中的自定义指标



## 操作

### 配置

#### 准备工作

下载prometheus适配器的yaml配置

```
https://github.com/stefanprodan/k8s-prom-hpa
```

查询相关指标是否在普罗米修斯中存在, container_network_receive_bytes_total

```
sum by (pod_name) (rate(container_network_receive_bytes_total{pod_name="bow-hotvideo-recom-1366-b5558f5fd-j49xg"}[864s])) * 8 /1024/1000
```



#### 修改配置

```
# 生成cm-adapter-serving-certs.yaml文件（证书文件）
make certs

cd custom-metrics-api

# 默认相关pod及configmap等文件都会创建在montioring命名空间，这里修改为kube-system
sed -i 's/namespace: monitoring/namespace: kube-system/g' *.yaml

# 修改k8s-prometheus-adapter-amd64的镜像地址，因为默认的quay.io在国内访问不到
sed -i 's@quay.io/coreos/k8s-prometheus-adapter-amd64:v0.4.1@quay.azk8s.cn/coreos/k8s-prometheus-adapter-amd64:v0.4.1@g' custom-metrics-apiserver-deployment.yaml

# 修改需要连接的prometheus地址，改为集群中部署的prometheus的svc地址
sed -i 's@http://prometheus.monitoring.svc:9090@http://prometheus-prometheus-server.default.svc:9090@g' custom-metrics-apiserver-deployment.yaml




```

#### 应用配置

```
# 安装

kubectl apply -f ./
```



### 关键配置说明

Prometheus与K8S转换的关键配置在此文件中: 

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: kube-system
data:
  config.yaml: |
    rules:
    - seriesQuery: '{__name__=~"^http_requests_total$",kubernetes_namespace!="",kubernetes_pod_name!=""}'
      seriesFilters: []
      resources:
        overrides:
          kubernetes_namespace:
            resource: namespace
          kubernetes_pod_name:
            resource: pod
      name:
        matches: ^(.*)_total$
        as: ""
      metricsQuery: sum by (<<.GroupBy>>) (rate(<<.Series>>{<<.LabelMatchers>>}[1m]))
    - seriesQuery: '{__name__=~"^container_network_.*_total$",container_name="POD",namespace!="",pod_name!=""}'
      seriesFilters: []
      resources:
        overrides:
          namespace:
            resource: namespace
          pod_name:
            resource: pod
      name:
        matches: ^(.*)_bytes_total$
        as: "${1}_kbps"
      metricsQuery: sum by (<<.GroupBy>>) (rate(<<.Series>>{<<.LabelMatchers>>,container_name="POD"}[1m]))*8/1024/1000
    
...
```

其中, rules语法一开始看不懂, 可以根据官方文档说明查看, 这里简单解释一下:

这个配置文件主要干了以下几个活：

1. 通过`seriesQuery`从prometheus中获取相应的自定义指标，这会获取到多个指标
2. 通过`seriesFilters`过滤掉不需要的指标
3. 通过resources将prometheus中获取到的指标的namespace标签映射为k8s的namepace标签，将prometheus中获取到指标的`pod_name`标签映射为k8s的pod标签
4. 使用`metricsQuery`查询表达式获取k8s真正需要的指标
5. 通过name中的matches正则表达式来匹配查询到的指标，并将其重命名为as中的值。在我们的示例里，即将`container_network_receive_bytes_total`给重命名为`container_network_receive_kbps`。需要说明的是，如果as为空，即代表删除name中的非括号部分的字段。

我们说过，prometheus-adapter的作用就是将prometheus中查询的指标，翻译成k8s能识别的监控指标，而这部分工作即由其配置文件来实现。

在`metricsQuery`的语句中，还使用到了几个变量：

- <<.GroupBy>>： 基于查询的指标分组，我们的示例里，即`pod_name`
- <<.Series>>：查询的指标名，在我们的示例里即`container_network_receive_bytes_total`
- <<.LabelMatchers>>： 匹配到的标签，在我们的实例里，即`namespace!="",pod_name!=""`

详细指标配置说明，可参考[k8s-prometheus-adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter)这个项目。

相关配置项文档： https://github.com/DirectXMan12/k8s-prometheus-adapter/blob/master/docs/config-walkthrough.md



### 测试

验证 指标是否存在

```
 # 验证custom metric server
 kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
 # 验证metric server
 kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes" | jq .
```

验证 指标的值是否存在

```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/bigdata/pods/ocean-biz-service-business-731-67957c7dbf-crvmm/container_network_receive_kbps" | jq .

...
    {
      "describedObject": {
        "kind": "Pod",
        "namespace": "default",
        "name": "ocean-biz-service-business-731-67957c7dbf-crvmm",
        "apiVersion": "/v1"
      },
      "metricName": "container_network_receive_kbps",
      "timestamp": "2019-10-21T08:16:32Z",
      "value": "49152"
    }
...

```

### 服务使用

创建弹性伸缩的Yaml即可, 容器平台中有对应的图形化操作界面, 此处直接贴配置

```
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: auto-ocean-biz-service-business-731
  namespace: bigdata
  labels:
    appDeploymentId: "731"
    appId: "679"
    autoCreate: "true"
    name: auto-ocean-biz-service-business-731
spec:
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: ocean-biz-service-business-731
  minReplicas: 1
  maxReplicas: 3
  metrics:
  - type: Pods
    pods:
      metricName: container_network_receive_kbps
      targetAverageValue: 80m
```

应用上述配置后, 服务就可以根据`流量`来弹性扩缩容了, 我们可以通过`ab `测试来给服务提高流量负载, 观察其扩容状况

```
yum install httpd-tools –y
ab -n 1000 -c 5 '实例接口'
```

可以describe相关hpa, 来观察伸缩的具体情况: 

```
{
  ....
  "minReplicas":1,
  "maxReplicas":2,
  "metrics":[
   {
    "type":"Resource",
    "resource":{
     "name":"cpu",
     "targetAverageUtilization":10
    }
   }
  ]
 },
 "status":{
  "currentReplicas":1,
  "desiredReplicas":0,
  "conditions":[
   {
    "type":"AbleToScale",
    "status":"True",
    "lastTransitionTime":"2019-10-21T13:36:17Z",
    "reason":"SucceededGetScale",
    "message":"the HPA controller was able to get the target's current scale"
   },
   {
    "type":"ScalingActive",
    "status":"False",
    "lastTransitionTime":"2019-10-21T13:36:17Z",
    "reason":"FailedGetResourceMetric",
    "message":"the HPA was unable to compute the replica count: missing request for cpu on container cradle in pod bigdata/cradle-943-5d44bd4499-lxrm8"
   }
  ]
 }
}
```





## 优化

### 加速扩缩速度

在kube-apiserver的启动指令中添加如下参数：

```
kube-apiserver
  ...
  --requestheader-client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --requestheader-allowed-names=aggregator \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file=/etc/kubernetes/ssl/metrics-server.pem \
  --proxy-client-key-file=/etc/kubernetes/ssl/metrics-server-key.pem \
```

在kube-controller-manager的启动指令中添加如下参数：

```
kube-controller-manager
  ...
  --horizontal-pod-autoscaler-use-rest-clients=true \
  --horizontal-pod-autoscaler-downscale-delay=5m0s \
  --horizontal-pod-autoscaler-upscale-delay=1m0s \
  --horizontal-pod-autoscaler-sync-period=20s \
  ...
```

配置项说明：

- horizontal-pod-autoscaler-use-rest-clients： 开启基于rest-clients的自动伸缩

- horizontal-pod-autoscaler-sync-period：自动伸缩的检测周期为20s，默认为30s

- horizontal-pod-autoscaler-upscale-delay：当检测到满足扩容条件时，延迟多久开始缩容，即该满足的条件持续多久开始扩容，默认为3分钟

- horizontal-pod-autoscaler-downscale-delay：当检测到满足缩容条件时，延迟多久开始缩容，即该满足条件持续多久开始缩容，默认为5分钟

  



### 指标选择

因为QPS指标需要应用暴露相关接口提供, 有一定侵入性, 因此采用了`container_network_receive_kbps`, 这个容器指标, 代表容器接入流量值, 单位为Kbps, 后期有QPS需求可以添加QPS相关指标!

此指标正好对应`容器平台` --> `实例详情` --> `网络(图标)` --> `上行带宽` 的值







## 其他

### 参考链接

- 减少扩容间隔: 

  https://github.com/gtirloni/kubernetes-website/blob/master/docs/tasks/run-application/horizontal-pod-autoscale.md

  https://rancher.com/blog/2018/2018-08-06-k8s-hpa-resource-custom-metrics/

  

- 搭建自定义指标框架

  https://www.yangcs.net/posts/custom-metrics-hpa/

  https://github.com/stefanprodan/k8s-prom-hpa

  https://github.com/gtirloni/kubernetes-website/blob/master/docs/tasks/run-application/horizontal-pod-autoscale.md

- 原理

  https://zhuanlan.zhihu.com/p/34555654
  
  https://xuchao918.github.io/2019/01/10/如何实现K8s-Pod自定义指标弹性伸缩/

### 踩坑

单位换算

> 默认情况下，k8s中采集到的指标会在prometheus指标的值上乘1000，然后加个m的单位，可以直接理解为k8s中的1000m即prometheus中的1
>
> 

指标与应用创建先后顺序

> 注： 在我实际测试当中，如果先创建应用，后在k8s中生成指标（在prometheus-adapter的configmap中配置指标映射），则hpa无法正常拿到该应用的自定义指标，无法基于其弹性伸缩。当先生成指标，后创建应用时，则正常。还待进一步测试验证。

