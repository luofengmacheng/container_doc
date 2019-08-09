## k8s架构与功能

### 1 k8s架构

k8s的架构跟一般的分布式系统没有太大的区别，基本是上server-agent模式：

server上的组件包含：

* api-server 其它组件之间的交互都通过api-server实现
* scheduler 调度器，将应用调度到合适的节点上启动
* controller manager 控制器

agent(工作节点)上的组件包含：

* kubelet 相当于k8s的agent，负责管理工作节点的容器
* kube-proxy 负责服务的负载均衡
* 容器 Docker或者rkt

### 2 k8s提供的功能

k8s提供的功能基本都是围绕运维集群过程中的问题进行的：

* 服务发现

服务是提供相同功能的实例组成的集合，对外提供统一的功能，如果可以通过同一个标识符访问该服务，那么该服务内部发生了什么问题，对方是不用关心的，便于容灾和负载均衡。

* 自动扩缩容(维持一定数量的副本数)

如果是接入层这种无状态的pod提供的服务，这种pod可以运行在任何机器上，运维可以自定义pod的数量，k8s可以根据一定的策略将pod调度到相应的节点进行部署，k8s自身也可以根据一定的策略自动进行扩缩容。

* 负载均衡

为了能够将请求分发到多台机器上，负载均衡是运维必须要面对的问题。而kubernetes原生就可以通过kube-proxy提供负载均衡功能。

* 滚动更新(灰度发布)

为了在升级版本过程中不影响上游系统的访问，k8s可以在升级过程中自动隔离不可用的pod。

* 健康检查和自我修复

kubernetes可以自动检测pod的健康度，当pod由于节点的原因不可用时，k8s可以自动创建pod自我修复。