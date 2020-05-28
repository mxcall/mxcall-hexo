---

title: Flink1.10Container环境实战(转载)

date: 2020-05-28 08:00:00

toc: true

categories: [工作, 程序员, flink]

tags: [flink, k8s, docker]



---

# Flink1.10Container环境实战(转载)

本文转载自Flink社区文章

https://mp.weixin.qq.com/s?__biz=MzU3Mzg4OTMyNQ==&mid=2247487204&idx=1&sn=eec328dbb96b0073b69065231cf44622&chksm=fd3b80a6ca4c09b02b6ec6e55c3cbc0baef31675da6fd5f18b271bb9171c9dc6043cde41e672&scene=126&sessionid=1590627458&key=298423b212b807ab334dbfe819fadba1441f1713cac82583c1f0328ac2ecedea1f9b8c624e2632ddbd386822510ddccfc60d883f8351de783f76408927a08233df207e242abb1c77681e06ef68b3e322&ascene=1&uin=MzA5NzMxMzE1&devicetype=Windows+10+x64&version=62090070&lang=zh_CN&exportkey=A1r0MT3Fcvweqjg2xLSrBro%3D&pass_ticket=QxCu6ZQ0cQDqs8n7%2F4%2FqzmQ4TfvCcyVfMwRwl9PwsZ7PoLYBoUCK5O3Ldgo9gmQ7



本文第一部分将简明扼要地介绍容器管理系统的演变；第二部分是 Flink on K8S 简介，包括集群的部署模式调度原理等等；第三部分是我们这一年以来关于 Flink on K8S 的实战经验分享，介绍我们遇到的问题、踩过的坑；最后一部分是 Demo，将手把手演示集群部署、任务提交等等。





** 容器管理系统的演变 **

![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528112801.webp)





首先是以一个 Kubernetes 非内核开发人员的角度去探讨其和 YARN 之间的关系。众所周知，Apache Hadoop YARN 可能是在国内用途最广的一个调度系统，主要原因在于 Hadoop HDFS 在国内或者是在整个大数据业界，是一个使用最广泛的存储系统。因此，基于其上的 YARN 也自然而然成为了一个广为使用的一个调度系统，包括早期的 Hadoop MapReduce。随着 YARN 2.0 之后 Framework 的开放，Spark on YARN 以及 Flink on YARN 也可以在 YARN 上进行调度。



当然 YARN 本身也存在一定的局限性。



- 如资源隔离，因为 YARN 是以 Java 为基础开发的，所以它很多资源方面的隔离有一些受限。
- 另外对 GPU 支持不够，当然现在的 YARN 3.0 已经对 GPU 的调度和管理有一定支持，但之前版本对GPU 支持不是很好。



所以在 Apache 基金会之外，CNCF 基金会基于 Native Cloud 调度的 Kubernetes 出现了。



从开发人员角度来看，我认为 Kubernetes 是更像一个操作系统，可以做非常多的事情。当然这也意味着 Kubernetes 更复杂、学习曲线比较陡峭，你需要理解很多定义和概念。相比之下，YARN 主要管理资源调度部分，对整个操作系统而言，它体量要小很多。当然，不可置否，它也是一个大数据生态的先驱。接下来我会将焦点放在 Kubernetes 上面，探讨从 YARN 的 Container 向 Kubernetes 的 Container（或者 POD）的演变过程中，我们总结的经验和教训。



**Flink on K8S intro**





**部署集群**





![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528112847.webp)



上图展示了 Flink Standalone Session on K8S 上调度流程图，蓝色虚线框内是运行在 Kubernetes 集群内部组件，灰色框的是 Kubernetes 原生所提供的命令或组件，包括 kubectl 和 K8S Master。左侧罗列了 Flink 官方文档上提供的5个 yaml 文件，可以用来在 K8S 上部署最简单的 Flink Standalone Session 集群。



启动集群所需要执行的 kubectl 命令如下：



```
kubectl create -f flink-configuration-configmap.yaml
kubectl create -f jobmanager-service.yaml
kubectl create -f jobmanager-deployment.yaml
kubectl create -f taskmanager-deployment.yaml
```



- **首先**，它会向 K8S Master 申请创建 Flink ConfigMap，在 ConfigMap 中提供了 Flink 集群运行所需要的配置，如：flink-conf.yaml 和 log4j.properties；
- **其次**，创建 Flink JobManager 的 service，通过 service 来打通 TaskManager 和 JobManager之间的联通性；
- **然后**，创建 Flink Jobmanager 的 Deployment，用来启动 JobMaster，包含的组件有 Dispatcher 和 Resource manager。
- **最后**，创建 Flink TaskManager 的 Deployment，用来启动 TaskManager，因为 Flink 官方 taskmanager-deployment.yaml 示例中指定了2个副本，所以图中展示了2个 TM 节点。



另外，还有一个可选操作是创建 JobManager REST service，这样用户就可以通过REST service 来提交作业。



以上就是 Flink Standalone Session 集群的概念图。





**作业提交**



下图展示了使用 Flink client 向该 Standalone Session 集群提交作业的流程细节。



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528112906.webp)





使用 Flink client 提交作业的命令是：



- 

```
./bin/flink run -m : ./examples/streaming/WordCount.jar
```



其中 -m 所需的参数 public-node-IP 和 node-port 正是通过 jobmanager-service.yaml 所暴露 REST service 的 IP 和端口。执行该命令就可以向集群提交一个 Streaming WordCount 作业。此流程与 Flink Standalone 集群所运行的环境无关，无论是运行在 K8S 之上，还是运行在物理机之上，提交作业的流程是一致的。



Standalone Session on K8S 的优缺点：



- 优点是无需修改 Flink 源码，仅仅只需预先定义一些yaml 文件，集群就可以启动，互相之间的通信完全不经过 K8S Master；
- 缺点是资源需要预先申请无法动态调整，而 Flink on YARN 是可以在提交作业时声明集群所需的 JM 和 TM 的资源。



因此社区在 Flink 1.10 进程中，也是我们阿里负责调度的同学，贡献的整个 native 计算模式的Flink on K8S，也是我们过去一年在实战中所总结出来的 Native Kubernetes。



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528112923.webp)





它最大的区别就是当用户通过 Flink client 提交作业时，整个集群的 JobMaster 通过 K8sResourceManager 向 K8S Master 动态申请资源去创建运行 TaskManager 的 POD，然后 TaskManager 再与 JobMaster 互相之间通信。有关 Native Kubernetes的细节请参考王阳所分享的《Running Flink on Kubernetes natively》。



总而言之，我们可以像使用 YARN 一样的去使用 K8S，相关的配置项也尽量做到与 YARN 类似。不过为了方便讲解，接下来我会使用 Standalone Session集群来展示，而下文介绍的部分功能，在 Flink 1.10 还未实现，预计在 Flink 1.11 完成。



**Flink on K8S 实战分享**







**日志搜集**



当我们在 Flink on K8S 上运行一个作业，有一个功能性问题无法回避，就是日志。如果是运行在 YARN 上，YARN 会帮我们做这件事，例如在 Container 运行完成时，YARN 会把日志收集起来传到 HDFS，供后期查看。但是 K8S 并未提供日志搜集与存储，所以我们可以有很多选择去做日志（收集、展示）的事情。尤其是当作业因为异常导致 POD 退出，POD 退出后日志会丢失，这将导致异常排查变得非常困难。



如果是 YARN，我们可以用命令 yarn logs -applicationId <applicationId> 来查看相关日志。但是在 K8S 上怎么办？



目前我们比较常见的做法是使用 fluentd 来搜集日志，且已经在部分用户生产环境有所应用。



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528112957.webp)





fluentd 也是一个 CNCF 项目，通过配置一些规则，比如正则匹配，就可以将 logs 目录下的*.log 、*.out 及 *.gc 日志定期的上传到 HDFS 或者是其他分布存储文件系统，以此来解决我们的日志收集功能。这也意味着在整个 POD 的里面，除了 TM 或 JM 之外，我们需要再启动一个运行着 fluentd 进程的 Container（sidecar）。



当然，还有其他办法，比如一个不需要再增加 Container 的方式：我们可以使用logback-elasticsearch-appender 将日志发到 Elasticsearch。其实现原理是通过Elasticsearch REST API 支持的 scoket stream 方式，将日志直接写入Elasticsearch。



相比于之前的 fluentd 来说，优点是不需要另启一个 Container 来专门收集日志，缺点是无法搜集非 log4j 日志，比如 System.out、System.err 打印的日志，尤其是作业发生 core dump，或者发生 crash 时，相关日志会直接刷到System.out、System.err 里面。从这个角度来看使用 logback-elasticsearch-appender 写入 Elasticsearch 的方案也不是那么完美了。相比之下，fluentd 可以自由地配置各式各样的策略来搜集所需要的日志信息。





**Metrics**



日志可以帮助我们观察整个作业运行的情况，尤其是在出问题之后，帮助我们回溯场景，进行一些排查分析。另外一个老生常谈也非常重要的问题就是 Metrics 和监控。业界已经有很多种监控系统解决方案，比如在阿里内部使用比较多的 Druid、开源InfluxDB 或者商用集群版 InfluxDB、CNCF 的 Prometheus 或者 Uber 开源的 M3 等等。



然后我们这里直接拿 Prometheus 进行讨论，因为 Prometheus 与 Kubernetes 均属于 CNCF 项目，在指标采集领域具备先天优势，从某种程度上来说Prometheus 是 Kubernetes 的一个标配监控采集系统。Prometheus 可以实现功能很多，不仅可以去做报警，也可以定一些规则做定期的多精度管理。





![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113020.webp)



但是我们在实际生产中发现一个问题，Prometheus 的水平拓展支持不够好。大家可以看到上图右侧部分，Prometheus 所谓的联邦分布式架构其实就是多层结构，一层套一层，然后它上面节点负责路由转发去下一层查询结果。很明显，无论部署多少层，越往上的节点越容易成为性能瓶颈，而且整个集群的部署也很麻烦。从我们接触到的用户来说，在规模不是很大的时候，单点的 Prometheus 就可以承担绝大部分的监控压力，但是一旦用户规模很大，比如几百个节点的 Flink 集群，我们就会发现单点 Prometheus 会成了一个非常大的性能瓶颈，无法满足监控需求。



我们怎么做到呢？



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113037.webp)





我们首先对不同 Flink 作业的 metrics 做了一致性哈希，当然肯定不会是一个作业的metrics 只发了一个 Prometheus，而是面向作业里面不同 scope 的 metrics，Flink的 metrics 力度从大到小分别是：



- JobManager/TaskManager metrics
- Job metrics(checkpoint次数、size和fail次数)
- task metrics
- operator metrics(每秒处理多少条record、receive的bytes数目)。



现在方案是先根据不同的 scope 去做一致性哈希，发到不同的 Prometheus 上，之后再配合 Thanos (灭霸，对就是在《复仇者联盟3》里面打完响指后去种瓜的农夫)。我个人理解 Thanos 是一个可以支持分布式查询 Prometheus 的增强组件。所以整个 Prometheus 架构，演变成单个 Prometheus 实例所在的 container 会搭载一个 Thanos sidecar。



当然整个架构会导致一些限制，这个限制也是我们做一致性哈希的原因，是因为当 Thanos 与 Prometheus 所搭配部署时，如果有一段 metrics数据，因为某些原因导致它既在 Prometheus A 里面，也在 Prometheus B 里面，那么在 Thanos query 里边它会有一定规则，对数据进行 abandon 处理，即去掉一个以另外一个为准， 这会导致 UI 上 metrics 图表的线是断断续续的，导致体验很不友好，所以我们就需要一致性哈希，并通过 Thanos 去做分布式查询。



但是整个方案实际运行中还是有一些性能问题，为什么 Prometheus 在很多业务级 metrics 上去表现其实很不错，而在 Flink 或者是这种作业级别上，它表现的会有一些压力呢？其实很重要的一个原因是作业 metrics 变化是非常急剧的。相比于监控HDFS、Hbase，这些组件的指标是有限的、维度也不高。我们用一个查询场景来解释维度的概念，例如说我们要查询在某个 hosts 的某个 job 的某个 task 上所有的 taskmanager_job_task_buffers_outPoolUsage，这些所说的查询条件，也就是用 tag 去做查询过滤，那么就有一个问题是 Flink 的 taskAttempId，这个是一个非常不友好的一个 tag，因为它是一个 uuid 且每当作业发生 failover 的时候，taskAttempId 就会发生变化。



如果作业不断 failover，然后不停地持久化新的 tag 到 Prometheus，如果 Prometheus 后面接的 DB 需要对 tag 构建一个索引的话，那么索引的压力会非常大。例如 InfluxDB 的压力就会非常大，可能会导致整个内存 CPU 不可用，这样的结果非常可怕。所以，我们还需要借助于社区在 report 这边把一些高维度的 tag 过滤掉，有兴趣的同学可以关注下 FLINK-15110。





**性能**



**■ 网络性能**



我们先介绍 network 性能。无论你用 CNI 或者 Kubernetes 的网络化插件，不可避免的会出现网络性能损耗。比较常见的 flannel，在一些测试项目上会有百分之30左右的性能损耗。也不是很稳定，我们使用时发现作业经常会报PartitionNotFoundException: Partition xx@host not found，也就是下游无法获取到上游的 Partition。



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113103.webp)



当然，你可以在 Flink 层去增大一些网络容错性，例如把 taskmanager.network.request-backoff.max 调到300秒，默认是10秒，然后把akka 的 timeout 调大一点。



还有一个让我们特别头疼的问题：





![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113124.webp)



我们发现作业运行过程中经常遇到 Connection reset by peer 问题，原因是 Flink 在设计时，对网络的稳定要求很高。因为要保证 Exactly once，如果数据传输失败，那么 Flink 就要 fail 整个 task 并重新启动，然后我们会发现经常会出现令人头疼的Connection reset by peer 问题，我们有几个的解决方案方式：



- 不要有异构网络环境（尽量不要跨机房访问）
- 云服务商的机器配置网卡多队列 (将实例中的网络中断分散给不同的CPU处理，从而提升性能)
- 选取云服务商提供的高性能网络插件：例如阿里云的 Terway
- Host network，绕开 k8s 的虚拟化网络（需要一定的开发量）



第一个要排查的问题就是集群不要有异构网络环境，因为有可能 Kubernetes 的宿主机在不同机房，然后跨机房访问遇到网络抖动的时候都就会比较头疼。然后是云服务商机器配置网卡多队列，因为 ECS 虚拟机，它是需要耗一定的 CPU 去做网络虚拟化。那么如果网卡不配置成多队列的话，有可能网卡只用了1到2个 core，然后虚拟化会把这2个 core 用光，用光的话会导致丢包，也就会遇到这种比较头疼的Connection reset by peer 问题。



还有一种方案是选取云服务商提供的高性能网络插件，当然这需要云服务商支持，比如阿里云的 Terway，Terway 对外描述是可以支持与 host network 一样的性能，而不是像 flannel 会带来一定的性能损耗。



最后一种，如果无法使用 Terway，我们可以用 host network 来绕开 K8S 虚拟化网络。不过这种方案首先是对 Flink 有一些开发工作，其次是如果你已经使用了Kubernetes，却还要使用 host network，从某种意义上来说，有一点奇怪，很不符合 K8S style。当然我们也在一些无法用 Terway 的机器，然后又遇到这个头疼的问题是，也提供了相应工程，部署时采用 host network，而不是使用 overlay 的flannel 方案。



**■ 磁盘性能**



接下来谈磁盘性能，前文提到过：所有虚拟化的东西都会带来一些性能损耗。对于 RocksDB 需要读写本地磁盘的场景，就很头疼，因为 overlay 的 file system 会有大概30%的性能损耗。



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113150.webp)





那怎么办呢？我们选择一种方式，就是使用 hostPath。简单来说，POD 可以访问到宿主机的物理盘。上图右侧部分就定义了 hostPath，当然需要预先保证 Flink 镜像的用户是具备访问宿主机目录权限，所以最好把这里目录改成 777 或者 775。



大家如果想用这个功能的话，可以查看 Flink-15656，它提供一个 POD 的 template，意味着你可以自行调整。因为我们知道 K8S 的功能特别多，特别繁杂，Flink 不可能为了每一个功能都去做个适配。你可以在 template 里面，比如定义 hostPath，然后你所写 POD 的就可以基于这个模板下面的 hostPath 就可以去访问目录了。





**OOM killed**



OOM killed 也是个比较头疼的问题。因为在容器环境下，部署服务的时候，我们需要预先设置 POD 所需 memory 和 CPU 的资源，然后 Kubernetes 会指定配置去相关 node (宿主机)上面申请调度资源。申请资源除了要设置 request 之外，还有会设置 limit——一般都会打开 limit——它会对申请的 memory 和 CPU 进行限制。



比如说 node 的物理内存是 64G，然后申请运行8个8G内存的 POD，看着好像没有问题，但是如果对这8个 POD的没有任何 limit的话，假如每个用到10G，那么就会导致 POD 之间出现资源竞争，现象是一个 POD 运行正常另外一个 POD 忽然被 Kill，所以就要做一个memory limit。memory limit 带来的问题是：POD 莫名其妙退出，然后查看 Kubernetes 的 event 发现是因为 POD 被 OOM killed 了。我相信如果用过Kubernetes 的同学肯定遇到过相关问题。



我们是怎么排查的呢？



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113209.webp)





第一个是我们可以在 JVM 端开启 native 内存追踪，可以定期去查看，但这只能看到 JVM 所申请的 native 内存，包括如 Metaspace，非 JVM 就无法分析了；还有一个就是万能的 Jemalloc 和 jeprof 去做定期 dump 内存进行分析。



老实说第2个功能我们可能用的比较少，因为我们以前在 YARN 上面会这样用，就是说发现有的作业内存很大，因为 JVM 对最大内存会做限制，所以肯定是 native 这边出的问题，那么到底是哪个 native 出问题，就可以 Jemalloc+jeprof 作内存分析。比如我们之前遇到过用户自己去解析 config 文件，结果每次都要解压一下，最后把内存撑爆了。



当然这是一种引起 OOM 的场景，但更多的可能是 RocksDB 引发 OOM，当然如果是使用了 RocksDB 这种省 native 内存的 state backend。所以我们在 Flink 1.10 做了一个功能贡献给社区，就是对 RocksDB 的 memory 进行管理，由参数state.backend.rocksdb.memory.managed 控制是否进行管理，默认是开启。

我们下面这个图是什么呢？





![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113224.webp)



是在 RocksDB 没有使用 memory 控制，这里一共定了4个 state，分别是 value、list、map 和 window，大家可以看到最顶端的线是 block cache usage 加上RocksDB 的 write buffer 就构成了 RocksDB 当前所使用总内存的大小。大家看到这4个加起来的话差不多超过400M了。



原因是 Flink 目前的 RocksDB 对 state 数没有限制，一个 state 的就是一个 Column Family，而 Column Family 就会额外独占所用的 write buffer 和 block cache。默认情况下，一个 Column Family 最多拥有2个64MB write buffer 和一个 8MB block cache，大家可以算一算，一个 state 就是136MB，四个 state 就是544MB。



如果我们开启了 state.backend.rocksdb.memory.managed，我们会看到4个 state所使用的 block cache 折线走势基本一致：



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113258.webp)





为什么呢？是因为实现了 cache share 功能。就是说，我们在一个 state 里面我们先创建一个 LRU cache，之后无论是什么情景都会从 LRU cache 里面去做内存的分发和调度，然后借助 LRU cache，最近最少被用的内存会被释放掉。所以在 Flink 1.10之后，我们说开启 state.backend.rocksdb.memory.managed 可以解决大部分问题。



![](https://gitee.com/matrixcall/bed001/raw/master/img/2020/05/20200528113317.webp)





但是，当然万事都有但是，我们开发过程中发现：RocksDB cache share 的功能做的不是特别好。这涉及到一些实现原理细节，比如没法去做 strict cache，如果你开启的话可能会碰到奇怪的NPE问题，所以说在某些特定场景下可能做的不是很好。这时你可能就要适当增大taskmanager.memory.task.off-heap.size 以提供更多的缓冲空间。



当然我们首先要知道它大概用多少内存空间。刚才我们展示的内存监控图里面，是需要打开参数 state.backend.rocksdb.metrics.block-cache-usage:true，打开之后，我们可以在 metrics 监控上面去获取到相关的指标，观察一下大概超用到多少。比如说1GB一个 state TM 默认的 manager 是 294MB。



所以说你发现比如说你可能超过很多，比如说偶尔会到300MB，或者310MB，你这时候就可以考虑配置参数taskmanager.memory.task.off-heap.size (默认是0)来再增加一部分内存，比如说再加64MB，表示在 Flink 所申请的 off-heap 里面再额外开辟出来一块空间，给RocksDB 做一段 Buffer，以免他被 OOM killed。这个是目前所能掌握的一个解决方案，但根本的解决方案可能需要跟 RocksDB 社区去一起去协同处理。



我们也希望如果有同学遇到类似问题可以跟我们进行交流，我们也非常乐意和你一起去观察、追踪相关问题。



**Demo**



最后一部分演示使用 hostPath 的 demo，大部分 yaml 文件与社区的示例相同，task manager 的部署 yaml 文件需要修改，见下：





```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-taskmanager
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flink
      component: taskmanager
  template:
    metadata:
      labels:
        app: flink
        component: taskmanager
    spec:
      containers:
      - name: taskmanager
        image: reg.docker.alibaba-inc.com/chagan/flink:latest
        workingDir: /opt/flink
        command: ["/bin/bash", "-c", "$FLINK_HOME/bin/taskmanager.sh start; \
          while :;
          do
            if [[ -f $(find log -name '*taskmanager*.log' -print -quit) ]];
              then tail -f -n +1 log/*taskmanager*.log;
            fi;
          done"]
        ports:
        - containerPort: 6122
          name: rpc
        livenessProbe:
          tcpSocket:
            port: 6122
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf/
        - name: state-volume
          mountPath: /dump/1/state
        securityContext:
          runAsUser: 9999  # refers to user _flink_ from official flink image, change if necessary
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
          - key: log4j.properties
            path: log4j.properties
      - name: state-volume
        hostPath:
          path: /dump/1/state
          type: DirectoryOrCreate
```



**Q&A 问答**



**1、Flink 如何在 K8S 的 POD 中与 HDFS 交互？**



其与 HDFS 交互很简单，只要把相关依赖打到镜像里面就行了。就是说你把 flink-shaded-hadoop-2-uber-{hadoopVersion}-{flinkVersion}.jar 放到 flink-home/lib目录下，然后把一些 hadoop 的配置比如 hdfs-site.xml、 core-site.xml 等放到可以访问的目录下，Flink 自然而然就可以访问了。这其实和在一个非 HDFS 集群的节点上，要去访问 HDFS 是一样的。



**2、Flink on K8S 怎么保证 HA？**



其实 Flink 集群的 HA 与是否运行在 K8S 之上没什么关系，社区版的 Flink 集群 HA需要 ZooKeeper 参与。HA 需要 ZooKeeper 去实现 checkpoint Id counter、需要ZooKeeper 去实现 checkpoint stop、还包括 streaming graph 的 stop，所以说HA 的核心就变成如何在 Flink on K8S 的集群之上，提供 ZooKeeper 的服务，ZooKeeper 集群可以部署在 K8S 上或者物理机上。同时社区也有尝试在 K8S 里面借用 etcd 去支持提供一套 HA 方案，目前真正工业级的 HA，暂时只有 zookeeper 这一种实现。



**3、Flink on K8S 和 Flink on YARN，哪个方案更优？怎样选择？**



Flink on YARN 是目前比较成熟的一套系统，但是它有点重，不是云原生（cloud native）。在服务上云的大趋势下，Flink on K8S 是一个光明的未来。Flink on YARN 是一个过去非常成熟一套体系，但是它在新的需求、新的挑战之下，可能缺乏一些应对措施。例如对很多细致的 GPU 调度，pipeline 的创建等等，概念上没有K8S 做得好。



如果你只是简单运行一个作业，在 Flink on YARN 上可以一直稳定运行下去，它也比较成熟，相比之下 Flink on K8S 够新、够潮、方便迭代。不过目前 Flink on K8S 已知的一些问题，比如学习曲线比较陡峭，需要一个很好的 K8S 运维团队去支撑。另外，K8S 本身虚拟化带来的性能影响，正如先前介绍的无论是磁盘，还是网络，很难避免一定的性能损耗，这个可能是稍微有点劣势的地方，当然相比这些劣势，虚拟化（容器化）带来的优点更明显。



**4、 /etc/hosts 文件如何配置的？我理解要跟 HDFS 交互，需要把 HDFS 节点 IP 和 host，映射写到 /etc/hosts 文件。**



通过通过 Volume 挂载 ConfigMap 内容并映射到 /etc/hosts 来解决，或者无需修改 /etc/hosts 转而依赖 CoDNS。



**5、Flink on K8S 故障排查困难，你们是怎么解决的？**



首先 Flink on K8S 与 Flink on YARN 的故障排查有什么区别呢？主要是 K8S 本身可能会有问题，这就是稍微麻烦的地方。K8S 可以认为是一个操作系统，可能有很多复杂的组件在里面。YARN 是一个用 Java 实现的资源调度器，这时更多是宿主机故障导致集群异常。面对 K8S 可能出问题，我个人感觉是相比 YARN 来说要难查一些。因为它有很多组件，可能 DNS 解析出问题，就需要去查看 CoDNS 日志；网络出问题或者是磁盘出问题你要去查看 kube event；POD 异常退出，需要去查看 event POD 退出的原因。实话实话，确实需要一定的门槛，最好是需要运维支持。



但如果是说 Flink 故障排查，这在 K8S 或是 YARN 排查手段都一样，



- 查看日志，检测是否有 exception;
- 如果是性能就需要用 jstack，查看 CPU、调用栈卡在哪里;
- 如果发现总是有 OOM 风险，或者老年代总是打的很满，或者 GC 频繁，或者Full GC 导致 Stop the world，就需要 jmap 查看哪块占内存，分析是否存在内存泄露



这些排查方法是与平台是无关的，是一个放之四海而皆准的排查流程。当然需要注意POD 镜像中可能会缺少一些 Debug 工具，所以建议大家在搭建 Flink on K8S 集群时，构建私有镜像，在构建的过程中安装好相应的 Debug 工具。





