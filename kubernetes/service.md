## kubernetes中的服务注册和服务发现

### 1 服务注册和服务发现

什么是服务？为什么会有服务，不能直接用IP和端口吗？

服务可以看做是一个标识，其它业务在访问该服务时可以在配置文件中配置该标识，而不用配置IP和端口。如果直接配置IP和端口，那么在对设备进行调整或者裁撤时，就需要修改其它业务的相关配置，在一个十分负载的系统中，这样的行为是不可接受的。

服务注册和服务发现：

其实可以理解为路由注册和路由发现。服务注册：在一个统一的服务管理中心进行注册，告知某个服务的标识和服务中的机器。服务发现：从服务管理中心获取一个服务的标识，将请求发送给该标识，由系统转发到实际处理请求的机器，或者通过标识获取服务的机器，然后再将请求发送给获取的机器。

### 2 服务注册(创建服务)

通过上面对服务的介绍，知道服务至少需要两个内容：服务名(服务标识号)和端口。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: love-test
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx-rs-test
```

该服务的服务名是nginx-service，服务的端口是80，映射到后端实际pod的端口号也是80，通过`app=nginx-rs-test`选择后端的pod。

上面的服务创建过程有个问题：如果服务后端的pod的端口变更时，就需要调整服务中端口映射关系。为了解决这个问题，在设置端口时可以对端口命名：

* 在创建pod时需要设置容器的端口和对应的名字
* 服务的端口映射关系中，映射的目的端口是端口名字而不是具体的端口

### 3 服务发现

服务发现即通过某种方式访问服务，根据服务的访问方向，在k8s中有三种方式：

* 在`集群内部`连接`集群内部`的服务
* 在`集群内部`连接`集群外部`的服务
* 在`集群外部`连接`集群内部`的服务

当然，就不会有在集群外部连接集群外部的服务了，不过这种方式也可以通过k8s做下代理。

#### 3.1 在`集群内部`连接`集群内部`的服务

1 通过ClusterIP

前面也介绍过服务的几种类型，默认创建的服务的类型是`ClusterIP`，也就是创建该服务后，k8s会给该服务分配一个内部的IP地址：ClusterIP，ClusterIP就相当于是该服务的标识号，之后就可以通过ClusterIP和服务的端口号进行访问了。

通过yaml文件创建服务后，可以通过`kubectl get svc`查看服务的ClusterIP。

2 通过环境变量

创建服务后，k8s会在pod的环境变量中加上服务的IP和端口：`NGINX_SERVICE_SERVICE_HOST`和`NGINX_SERVICE_SERVICE_PORT`。那么，就可以直接使用这两个变量作为服务的标识，本质上来说，这种方式也是通过ClusterIP进行访问的，只是通过一个固定标识去访问，因为ClusterIP也可能会改变。

3 通过FQDN(全限定域名)

k8s在启动容器后会修改容器的/etc/resolv.conf，其中主要有两个配置：

* nameserver dns域名服务器
* search 定义域名的搜索列表，default名字空间的容器的search会配置为`default.svc.cluster.local svc.cluster.local cluster.local`

FQDN其实就是域名而已，而在k8s中服务也可以通过域名访问，例如，上面创建的服务可以通过域名`nginx-service.love-test.svc.cluster.local`进行访问，可以登录到容器用curl进行测试。由于配置了search就可以直接用nginx-service作为域名访问。

search的作用是：当访问一个域名访问不到时就会尝试在search后面的域名下访问，例如，将nginx-service作为域名访问时，由于无法解析成IP地址，就会访问nginx-service.love-test.svc.cluser.local。

当然，用FQDN也有个问题：如果服务使用的不是默认端口，就需要使用其它方式获取服务的端口，例如，环境变量。

#### 3.2 在`集群内部`连接`集群外部`的服务

在集群内部访问集群外部的服务时，需要通过k8s将外部的实例映射成k8s内部的服务，这需要借助Endpoint对象完成。

Endpoint对象相当于是服务中的实例的列表，使用`kubectl describe service nginx-service`可以查看服务的Endpoint，Endpoint列表中就是每个实例的IP和端口号。当调用一个服务时，服务不是直接将请求转发给后端的机器，而是从Endpoint列表中获取一个实例进行转发。因此，只要将外部服务的实例填充到集群内部的服务的Endpoint，就可以通过服务进行转发。

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 80
```

创建一个服务external-service，其中不配置selector，那么k8s就不会为该服务创建Endpoint。

``` yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
    - ip: 1.1.1.1
    - ip: 2.2.2.2
    ports:
    - port: 80
```

然后，我们创建一个Endpoint，其中主要的配置就是addresses，用于配置外部的实例的IP，那么Endpoint如何跟Service进行关联呢？其实就是通过名字关联的，也就是它们的metadata.name属性是一样的，这里是external-service。

除了使用Endpoint，对于外部域名的访问还可以在k8s集群中创建一个服务用于转发，这种方式的好处是将外部的域名访问隐藏在服务后面，之后服务的内容变更后，调用方不用修改，相当于加了一层代理：

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  type: ExternalName
  externalName: api.xxx.com
  ports:
  - port: 80
```

#### 3.3 在`集群外部`连接`集群内部`的服务

服务的类型除了上面的ClusterIP，还有另外的三种类型：

* NodePort 每个集群节点都相当于是服务的代理，每个集群节点都会打开一个端口，当访问某个节点的该端口时，k8s会将流量重定向到Endpoint中的其中一个实例。那么，就可以通过三种方式访问后端的实例：通过ClusterIP的方式；通过pod的IP和端口；通过集群的节点和专用端口。
* LoadBalancer 这种方式是NodePort的一种扩展，会在每个节点前端再添加一个负载均衡器，但是这种方式通常需要云厂商支持才行，这里不做过多介绍
* Ingress

1 NodePort

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: love-test
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 80
    nodePort: 31234
  selector:
    app: nginx-rs-test
```

跟之前ClusterIP的不同点在于service.spec.type和service.spec.ports.nodePort，将类型设置为NodePort，而nodePort设置为31234，当然，nodePort也可以不配置，那么k8s就会随机选择一个端口。该服务就可以通过以下方式访问：

```
1 ClusterIP
curl ClusterIP:8080
2 NodePort
curl kubenode:31234
```

2 Ingress

Ingress相比于前面的NodePort和LoadBalancer的主要区别在于：

* 虽然NodePort可以通过节点对服务进行负载均衡，但是还是需要在节点前面额外添加一层负载均衡器以应对节点的异常，实际中不会使用这种方式
* LoadBalancer需要云平台的支持，导致与云平台有一定的耦合，并且每个服务都需要负载均衡器以及共有IP

Ingress是k8s原生支持的负载均衡器，并且能够代理多种服务，是实际使用最多的提供服务的方式。

### 4 探针(Probe)

前面已经介绍过存活探针，其实在k8s中还有另一种探针，用于表明容器已经准备好接收请求的就绪探针。存活探针和就绪探针在配置的使用过程基本一样：

* 都有HTTP、TCP、EXEC三种类型
* 都属于containers下面的一个字段：存活探针是livenessProbe，就绪探针是readinessProbe

只是它们的运作方式有些不一样：

* 存活探针探测失败表明服务已经不能正常工作；就绪探针探测失败表明该容器暂时还不能提供服务
* 如果存活探针探测失败，k8s会杀死容器并创建新的容器；如果就绪探针探测失败，k8s只是会将容器的IP从服务的Endpoint中移除