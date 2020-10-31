

# 二、基础集群部署 - kubernetes-simple

## 1. 部署ETCD（主节点）
#### 1.1 简介
&emsp;&emsp;kubernetes需要存储很多东西，像它本身的节点信息，组件信息，还有通过kubernetes运行的pod，deployment，service等等。都需要持久化。etcd就是它的数据中心。生产环境中为了保证数据中心的高可用和数据的一致性，一般会部署最少三个节点。我们这里以学习为主就只在主节点部署一个实例。
> 如果你的环境已经有了etcd服务(不管是单点还是集群)，可以忽略这一步。前提是你在生成配置的时候填写了自己的etcd endpoint哦~

#### 1.2 部署
**etcd的二进制文件和服务的配置我们都已经准备好，现在的目的就是把它做成系统服务并启动。**

```bash
#把服务配置文件copy到系统服务目录
$ cp ~/kubernetes-starter/target/master-node/etcd.service /lib/systemd/system/
#enable服务
$ systemctl enable etcd.service
#创建工作目录(保存数据的地方)
$ mkdir -p /var/lib/etcd
# 启动服务
$ service etcd start
# 查看服务日志，看是否有错误信息，确保服务正常
$ journalctl -f -u etcd.service
```

## 2. 部署APIServer（主节点）
#### 2.1 简介
kube-apiserver是Kubernetes最重要的核心组件之一，主要提供以下的功能
- 提供集群管理的REST API接口，包括认证授权（我们现在没有用到）数据校验以及集群状态变更等
- 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）

> 生产环境为了保证apiserver的高可用一般会部署2+个节点，在上层做一个lb做负载均衡，比如haproxy。由于单节点和多节点在apiserver这一层说来没什么区别，所以我们学习部署一个节点就足够啦

#### 2.2 部署
APIServer的部署方式也是通过系统服务。部署流程跟etcd完全一样，不再注释
```bash
$ cp target/master-node/kube-apiserver.service /lib/systemd/system/
$ systemctl enable kube-apiserver.service
$ service kube-apiserver start
$ journalctl -f -u kube-apiserver
```

#### 2.3 重点配置说明
> [Unit]  
> Description=Kubernetes API Server  
> ...  
> [Service]  
> \#可执行文件的位置  
> ExecStart=/home/michael/bin/kube-apiserver \\  
> \#非安全端口(8080)绑定的监听地址 这里表示监听所有地址  
> --insecure-bind-address=0.0.0.0 \\  
> \#不使用https  
> --kubelet-https=false \\  
> \#kubernetes集群的虚拟ip的地址范围  
> --service-cluster-ip-range=10.68.0.0/16 \\  
> \#service的nodeport的端口范围限制  
>   --service-node-port-range=20000-40000 \\  
> \#很多地方都需要和etcd打交道，也是唯一可以直接操作etcd的模块  
>   --etcd-servers=http://192.168.1.102:2379 \\  
> ...  

## 3. 部署ControllerManager（主节点）
#### 3.1 简介
Controller Manager由kube-controller-manager和cloud-controller-manager组成，是Kubernetes的大脑，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态。
kube-controller-manager由一系列的控制器组成，像Replication Controller控制副本，Node Controller节点控制，Deployment Controller管理deployment等等
cloud-controller-manager在Kubernetes启用Cloud Provider的时候才需要，用来配合云服务提供商的控制
> controller-manager、scheduler和apiserver 三者的功能紧密相关，一般运行在同一个机器上，我们可以把它们当做一个整体来看，所以保证了apiserver的高可用即是保证了三个模块的高可用。也可以同时启动多个controller-manager进程，但只有一个会被选举为leader提供服务。

#### 3.2 部署
**通过系统服务方式部署**
```bash
$ cp target/master-node/kube-controller-manager.service /lib/systemd/system/
$ systemctl enable kube-controller-manager.service
$ service kube-controller-manager start
$ journalctl -f -u kube-controller-manager
```
#### 3.3 重点配置说明
> [Unit]  
> Description=Kubernetes Controller Manager  
> ...  
> [Service]  
> ExecStart=/home/michael/bin/kube-controller-manager \\  
> \#对外服务的监听地址，这里表示只有本机的程序可以访问它  
>   --address=127.0.0.1 \\  
>   \#apiserver的url  
>   --master=http://127.0.0.1:8080 \\  
>   \#服务虚拟ip范围，同apiserver的配置  
>  --service-cluster-ip-range=10.68.0.0/16 \\  
>  \#pod的ip地址范围  
>  --cluster-cidr=172.20.0.0/16 \\  
>  \#下面两个表示不使用证书，用空值覆盖默认值  
>  --cluster-signing-cert-file= \\  
>  --cluster-signing-key-file= \\  
> ...  

## 4. 部署Scheduler（主节点）
#### 4.1 简介
kube-scheduler负责分配调度Pod到集群内的节点上，它监听kube-apiserver，查询还未分配Node的Pod，然后根据调度策略为这些Pod分配节点。我们前面讲到的kubernetes的各种调度策略就是它实现的。

#### 4.2 部署
**通过系统服务方式部署**
```bash
$ cp target/master-node/kube-scheduler.service /lib/systemd/system/
$ systemctl enable kube-scheduler.service
$ service kube-scheduler start
$ journalctl -f -u kube-scheduler
```

#### 4.3 重点配置说明
> [Unit]  
> Description=Kubernetes Scheduler  
> ...  
> [Service]  
> ExecStart=/home/michael/bin/kube-scheduler \\  
>  \#对外服务的监听地址，这里表示只有本机的程序可以访问它  
>   --address=127.0.0.1 \\  
>   \#apiserver的url  
>   --master=http://127.0.0.1:8080 \\  
> ...  

## 5. 部署CalicoNode（所有节点）
#### 5.1 简介
Calico实现了CNI接口，是kubernetes网络方案的一种选择，它一个纯三层的数据中心网络方案（不需要Overlay），并且与OpenStack、Kubernetes、AWS、GCE等IaaS和容器平台都有良好的集成。
Calico在每一个计算节点利用Linux Kernel实现了一个高效的vRouter来负责数据转发，而每个vRouter通过BGP协议负责把自己上运行的workload的路由信息像整个Calico网络内传播——小规模部署可以直接互联，大规模下可通过指定的BGP route reflector来完成。 这样保证最终所有的workload之间的数据流量都是通过IP路由的方式完成互联的。
**calico 会使用hostname进行注册，一定要配置不同的hostname**
#### 5.2 部署
**calico是通过系统服务+docker方式完成的**
```bash
$ cp target/all-node/kube-calico.service /lib/systemd/system/
$ systemctl enable kube-calico.service
$ service kube-calico start
$ journalctl -f -u kube-calico
```
hostname不能相同，不然会出错，修改后hostname后，原来的节点不会自动删除，需要手动删除，报错信息为：Calico node 'baqcentos7' is already using the IPv4 address 10.244.78.204:
手动删除命令为,获取列表然后删除
```bash
calicoctl get nodes：calico node 列表
calicoctl delete node xxx: 删除 dead node （通过 hostname）
calicoctl node run: 在新节点上跑 calico node ，让新的 node 生效 （bgp peer）
calicoctl node status: 查看状态，确认没有问题
```
#### 5.3 calico可用性验证
**查看容器运行情况**
```bash
$ docker ps
CONTAINER ID   IMAGE                COMMAND        CREATED ...
4d371b58928b   calico/node:v2.6.2   "start_runit"  3 hours ago...
```
**查看节点运行情况**
calicoctl 命令找不到的时候需要把kbs的bin文件添加到$PATH里面
```bash
#在 vi /etc/profile 的最后一行添加
export PATH=${PATH}:/usr/bin/kubernetes-bins/
```
```bash
$ calicoctl node status
Calico process is running.
IPv4 BGP status
+---------------+-------------------+-------+----------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+---------------+-------------------+-------+----------+-------------+
| 192.168.1.103 | node-to-node mesh | up    | 13:13:13 | Established |
+---------------+-------------------+-------+----------+-------------+
IPv6 BGP status
No IPv6 peers found.
```
**查看端口BGP 协议是通过TCP 连接来建立邻居的，因此可以用netstat 命令验证 BGP Peer**
```bash
$ netstat -natp|grep ESTABLISHED|grep 179
tcp        0      0 192.168.1.102:60959     192.168.1.103:179       ESTABLISHED 29680/bird
```
**查看集群ippool情况**
```bash
$ calicoctl get ipPool -o yaml
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 172.20.0.0/16
  spec:
    nat-outgoing: true
```
#### 5.4 重点配置说明
> [Unit]  
> Description=calico node  
> ...  
> [Service]  
> \#以docker方式运行  
> ExecStart=/usr/bin/docker run --net=host --privileged --name=calico-node \\  
> \#指定etcd endpoints（这里主要负责网络元数据一致性，确保Calico网络状态的准确性）  
>   -e ETCD_ENDPOINTS=http://192.168.1.102:2379 \\  
> \#网络地址范围（同上面ControllerManager）  
>   -e CALICO_IPV4POOL_CIDR=172.20.0.0/16 \\  
> \#镜像名，为了加快大家的下载速度，镜像都放到了阿里云上  
>   registry.cn-hangzhou.aliyuncs.com/imooc/calico-node:v2.6.2  

## 6. 配置kubectl命令（任意节点）
#### 6.1 简介
kubectl是Kubernetes的命令行工具，是Kubernetes用户和管理员必备的管理工具。
kubectl提供了大量的子命令，方便管理Kubernetes集群中的各种功能。
#### 6.2 初始化
使用kubectl的第一步是配置Kubernetes集群以及认证方式，包括：
- cluster信息：api-server地址
- 用户信息：用户名、密码或密钥
- Context：cluster、用户信息以及Namespace的组合

我们这没有安全相关的东西，只需要设置好api-server和上下文就好啦：
```bash
#指定apiserver地址（ip替换为你自己的api-server地址）
kubectl config set-cluster kubernetes  --server=http://192.168.1.102:8080
#指定设置上下文，指定cluster
kubectl config set-context kubernetes --cluster=kubernetes
#选择默认的上下文
kubectl config use-context kubernetes
```
> 通过上面的设置最终目的是生成了一个配置文件：~/.kube/config，当然你也可以手写或复制一个文件放在那，就不需要上面的命令了。

## 7. 配置kubelet（工作节点）
#### 7.1 简介
每个工作节点上都运行一个kubelet服务进程，默认监听10250端口，接收并执行master发来的指令，管理Pod及Pod中的容器。每个kubelet进程会在API Server上注册节点自身信息，定期向master节点汇报节点的资源使用情况，并通过cAdvisor监控节点和容器的资源。
#### 7.2 部署
**通过系统服务方式部署，但步骤会多一些，具体如下：**
```bash
#确保相关目录存在
$ mkdir -p /var/lib/kubelet
$ mkdir -p /etc/kubernetes
$ mkdir -p /etc/cni/net.d

#复制kubelet服务配置文件
$ cp target/worker-node/kubelet.service /lib/systemd/system/
#复制kubelet依赖的配置文件
$ cp target/worker-node/kubelet.kubeconfig /etc/kubernetes/
#复制kubelet用到的cni插件配置文件
$ cp target/worker-node/10-calico.conf /etc/cni/net.d/

$ systemctl enable kubelet.service
$ service kubelet start
$ journalctl -f -u kubelet
```
#### 7.3 重点配置说明
**kubelet.service**
> [Unit]  
Description=Kubernetes Kubelet  
[Service]  
\#kubelet工作目录，存储当前节点容器，pod等信息  
WorkingDirectory=/var/lib/kubelet  
ExecStart=/home/michael/bin/kubelet \\  
  \#对外服务的监听地址  
  --address=192.168.1.103 \\  
  \#指定基础容器的镜像，负责创建Pod 内部共享的网络、文件系统等，这个基础容器非常重要：K8S每一个运行的 POD里面必然包含这个基础容器，如果它没有运行起来那么你的POD 肯定创建不了  
  --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/imooc/pause-amd64:3.0 \\  
  \#访问集群方式的配置，如api-server地址等  
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\  
  \#声明cni网络插件  
  --network-plugin=cni \\  
  \#cni网络配置目录，kubelet会读取该目录下得网络配置  
  --cni-conf-dir=/etc/cni/net.d \\  
  \#指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，--cluster-domain 指定域名后缀，这两个参数同时指定后才会生效  
 --cluster-dns=10.68.0.2 \\  
  ...  

**kubelet.kubeconfig**  
kubelet依赖的一个配置，格式看也是我们后面经常遇到的yaml格式，描述了kubelet访问apiserver的方式
> apiVersion: v1  
> clusters:  
> \- cluster:  
> \#跳过tls，即是kubernetes的认证  
>     insecure-skip-tls-verify: true  
>   \#api-server地址  
>     server: http://192.168.1.102:8080  
> ...  

**10-calico.conf**  
calico作为kubernets的CNI插件的配置
```xml
{  
  "name": "calico-k8s-network",  
  "cniVersion": "0.1.0",  
  "type": "calico",  
    <!--etcd的url-->
    "ed_endpoints": "http://192.168.1.102:2379",  
    "logevel": "info",  
    "ipam": {  
        "type": "calico-ipam"  
   },  
    "kubernetes": {  
        <!--api-server的url-->
        "k8s_api_root": "http://192.168.1.102:8080"  
    }  
}  
```


## 8. 小试牛刀
到这里最基础的kubernetes集群就可以工作了。下面我们就来试试看怎么去操作，控制它。
我们从最简单的命令开始，尝试一下kubernetes官方的入门教学：playground的内容。了解如何创建pod，deployments，以及查看他们的信息，深入理解他们的关系。
具体内容请看慕课网的视频吧：  [《Docker+k8s微服务容器化实践》][1]

### 8.1 集群相关命令

```shell
#查看集群详细信息
kubectl cluster-info
#查看能发布应用程序的所有节点
kubectl get nodes
#通过将标签gpu= true添加到其中 一 个节点来实现（只需从kubectl get nodes返回的列表中选择 一个）：
kubectl label node gke-kubia-85f6-node-Orrx gpu=true
#获取指定命名空间下的资源,默认default命名空间，kubectl get ns 可以获取所有命名空间
kubectl get xxx --namespace kube-system
```

**工作节点**

一个 pod 总是运行在 **工作节点**。工作节点是 Kubernetes 中的参与计算的机器，可以是虚拟机或物理计算机，具体取决于集群。每个工作节点由主节点管理。工作节点可以有多个 pod ，Kubernetes 主节点会自动处理在群集中的工作节点上调度 pod 。 主节点的自动调度考量了每个工作节点上的可用资源。

每个 Kubernetes 工作节点至少运行:

- Kubelet，负责 Kubernetes 主节点和工作节点之间通信的过程; 它管理 Pod 和机器上运行的容器。
- 容器运行时（如 Docker）负责从仓库中提取容器镜像，解压缩容器以及运行应用程序。

最常见的操作可以使用以下 kubectl 命令完成：

- **kubectl get** - 列出资源
- **kubectl describe** - 显示有关资源的详细信息
- **kubectl logs** - 打印 pod 和其中容器的日志
- **kubectl exec** - 在 pod 中的容器上执行命令

您可以使用这些命令查看应用程序的部署时间，当前状态，运行位置以及配置。



### 8.2 Deployment

*Deployment 负责创建和更新应用程序的实例*

*应用程序需要打包成一种受支持的容器格式，以便部署在 Kubernetes 上*

创建 Deployment 时, Kubernetes 添加了一个 **Pod** 来托管你的应用实例

创建 Deployment 后，Kubernetes master 将应用程序实例调度到集群中的各个节点上。创建应用程序实例后，Kubernetes Deployment 控制器会持续监视这些实例。 如果托管实例的节点关闭或被删除，则 Deployment 控制器会将该实例替换为群集中另一个节点上的实例。 **这提供了一种自我修复机制来解决机器故障维护问题。**

```shell
# kubectl create deployment命令创建一个名为kubernetes-bootcamp的应用，镜像从image里面获取
kubectl create deployment kubernetes-bootcamp --image=gcr.io/google-samples/kubernetes-bootcamp:v1
#获取所有应用程序
kubectl get deployments
#在主节点执行kubectl proxy 命令开启一个代理，然后就可以通过外部网络访问了
kubectl proxy

```

### 8.3 Pod

*Pod 是一组一个或多个应用程序容器（例如 Docker），包括共享存储（卷), IP 地址和有关如何运行它们的信息。*

*如果它们紧耦合并且需要共享磁盘等资源，这些容器应在一个 Pod 中编排。*

Pod 可能既包含带有 Node.js 应用的容器，也包含另一个不同的容器，用于提供 Node.js 网络服务器要发布的数据。Pod 中的容器共享 IP 地址和端口，始终位于同一位置并且共同调度，并在同一工作节点上的共享上下文中运行。

Pod是 Kubernetes 平台上的原子单元。 当我们在 Kubernetes 上创建 Deployment 时，该 Deployment 会在其中创建包含容器的 Pod （而不是直接创建容器）。每个 Pod 都与调度它的工作节点绑定，并保持在那里直到终止（根据重启策略）或删除。 如果工作节点发生故障，则会在群集中的其他可用工作节点上调度相同的 Pod。

```shell
# 获取所有pod ，可以使用-o 自定义输出格式
kubectl get pods
# 获取该标签下的pod，kubectl describe deployment中有标签值，Lables
kubectl get pods -l 标签值
# 确保使用单引号来圈引电nv, 这样bash shell才不会解释感叹号（译者注：感叹号在bash中有特殊含义， 表示事件指示器）。
#列出不包含改标签纸的pod
#同理， 我们也可以将pod与以下标签选择器进行匹配：• creation_rnethod!=rnanual选择带有creation_rnethod标签， 并且值不等于manual的pod• env in (prod, devel)选择带有env标签且值为prod或devel的pod• env notin  (prod, devel)选择带有env标签，但其值不是prod或devel的pod
kubectl get pods -l '!标签值'
#显示标签
kubectl get pods --show-labels
#添加标签
kubectl label pod $POD_NAME app=v1
#修改标签
kubectl label pod $POD_NAME app=v1 --overwrite
#看该Pod中有哪些容器以及用于构建这些容器的镜像，详细信息 重启原因等
kubectl describe pods
#可以按照一定的规则访问pod里面的容器程序，开启proxy，然后使用地址。$POD_NAME pod的名字
curl http://localhost:8001/api/v1/namespaces/default/pods/$POD_NAME/proxy/
#查看pod的容器日志 添加 --previous
kubectl logs $POD_NAME
#查看pod的环境
kubectl exec $POD_NAME env
#进入到pod的运行环境里面，exit 退出环境
kubectl exec -ti $POD_NAME bash
#删除pod kubia-gup,可以指定多个pod一起删除
kubectl delete pod kubia-gup
#可以使用标签选择器(creation_method=manual)删除
kubectl  delete  po -l  creation_method=manual
#通过命名空间删除pod，也会删除命名空间
kubectl  delete  ns custom-namespace
#你可能还记得我们使用kubectl run命令创建了第一个pod。在第2章中提过这不会直接创建pod,而是创建一个ReplicationCcontroller,然后再由ReplicationCcontroller创建pod。因此只要删除由该ReplicationCcontroller创建的pod,它便会立即创建一“个新的pod。如果想要删除该pod,我们还需要删除这个ReplicationCcontroller.
#kubectl delete all --all 命令也会删除名为 kubernetes 的Service,  但它应该会在几分钟后自动重新创建。
```

### 8.4 Service

*Kubernetes 的 Service 是一个抽象层，它定义了一组 Pod 的逻辑集，并为这些 Pod 支持外部流量暴露、负载平衡和服务发现。*

*你也可以在创建 Deployment 的同时用 `--expose`创建一个 Service 。*

Kubernetes 中的服务(Service)是一种抽象概念，它定义了 Pod 的逻辑集和访问 Pod 的协议。Service 使从属 Pod 之间的松耦合成为可能。 和其他 Kubernetes 对象一样, Service 用 YAML [(更推荐)](https://kubernetes.io/zh/docs/concepts/configuration/overview/#general-configuration-tips) 或者 JSON 来定义. Service 下的一组 Pod 通常由 *LabelSelector* (请参阅下面的说明为什么您可能想要一个 spec 中不包含`selector`的服务)来标记。

尽管每个 Pod 都有一个唯一的 IP 地址，但是如果没有 Service ，这些 IP 不会暴露在群集外部。Service 允许您的应用程序接收流量。Service 也可以用在 ServiceSpec 标记`type`的方式暴露

- *ClusterIP* (默认) - 在集群的内部 IP 上公开 Service 。这种类型使得 Service 只能从集群内访问。
- *NodePort* - 使用 NAT 在集群中每个选定 Node 的相同端口上公开 Service 。使用`:` 从集群外部访问 Service。是 ClusterIP 的超集。
- *LoadBalancer* - 在当前云中创建一个外部负载均衡器(如果支持的话)，并为 Service 分配一个固定的外部IP。是 NodePort 的超集。
- *ExternalName* - 通过返回带有该名称的 CNAME 记录，使用任意名称(由 spec 中的`externalName`指定)公开 Service。不使用代理。这种类型需要`kube-dns`的v1.7或更高版本。

另外，需要注意的是有一些 Service 的用例没有在 spec 中定义`selector`。 一个没有`selector`创建的 Service 也不会创建相应的端点对象。这允许用户手动将服务映射到特定的端点。没有 selector 的另一种可能是您严格使用`type: ExternalName`来标记。

Service 通过一组 Pod 路由通信。Service 是一种抽象，它允许 Pod 死亡并在 Kubernetes 中复制，而不会影响应用程序。在依赖的 Pod (如应用程序中的前端和后端组件)之间进行发现和路由是由Kubernetes Service 处理的。

Service 匹配一组 Pod 是使用 [标签(Label)和选择器(Selector)](https://kubernetes.io/zh/docs/concepts/overview/working-with-objects/labels), 它们是允许对 Kubernetes 中的对象进行逻辑操作的一种分组原语。标签(Label)是附加在对象上的键/值对，可以以多种方式使用:

- 指定用于开发，测试和生产的对象
- 嵌入版本标签
- 使用 Label 将对象进行分类

标签(Label)可以在创建时或之后附加到对象上。他们可以随时被修改。现在使用 Service 发布我们的应用程序并添加一些 Label 。

```shell
#获取所有service
kubectl get services
#暴露服务，将应用开发，然后在其他节点就可以通过 ip+映射端口访问了
kubectl expose deployment/kubernetes-bootcamp --type="NodePort" --port 8080
#查看service详细信息
kubectl describe services 
#通过标签删除服务
kubectl delete service -l run=kubernetes-bootcamp
```

创建一个名叫kubia的服务，它将在端口80接收请求并将连接路由到具有标签选择器是app = kubia的pod的8080端口上。kubia-svc.yaml

> 在创建一个有多个瑞口的服务的时候，必须给每个瑞口指定名字。
>
> 注意标签选择器应用于整个服务，不能对每个端口做单独的配置。如果不同的pod有不同的端口映射关系，需要创建两个服务。

```yaml
apiVersion: v1
kind: Service
metadata:
	name: kubia
sepc:
	sessionAffinity: ClientIP #使服务代理将来自同一个 client IP 的所有请求转发至同 一 个 pod上。默认是None,即轮询。
	ports:
	- name: http #pod的8080端口映射成80端口
	  port: 80 #该服务的可用端口
	  targetPort: 8080 #服务将连接转发到的容器端口
	- name: https #pod的8443端口映射成443端口
	  port: 443 #该服务的可用端口
	  targetPort: 8443 #服务将连接转发到的容器端口
	selector:
		app: kubia #具有 app=kubia 标签的pod都属于该服务
```

给端口使用别名

```yaml
apiVersion: v1
kind: Service
metadata:
	name: kubia
sepc:
	containers:
	- name: kubia
	  ports:
	  - name: http #端口8080被命名为http
	    containerPort: 8080
	  - name: https #端口8443被命名为https
	  	containerPort: 8443
	sessionAffinity: ClientIP #使服务代理将来自同一个 client IP 的所有请求转发至同 一 个 pod上。默认是None,即轮询。
	ports:
	- name: http #pod的http端口映射成80端口
	  port: 80 #该服务的可用端口
	  targetPort: http #服务将连接转发到的容器端口
	- name: https #pod的https端口映射成443端口
	  port: 443 #该服务的可用端口
	  targetPort: https #服务将连接转发到的容器端口
	selector:
		app: kubia #具有 app=kubia 标签的pod都属于该服务
```

> 注意pod 是否使用内部的DNS服务器是根据pod中spec的dnsPolicy属性来决定的。
>
> backend-database.default.svc.cluster.local
>
> backend-database对应于服务名称，default 表示服务在其中定义的名称空间，而svc.cluster.local是在所有集群本地服务名称中使用的可配置集群域后缀。在请求的URL中，可以将服务的名称作为主机名来访问服务。因为根据每个pod容器DNS解析器配置的方式，可以将命名空间和svc.cluster.local后缀省略掉。查看一下容器中的**/etc/resilv.conf**文件就明白了。
>
> 无法ping通服务IP的原因：curl这个服务是工作的，但是却ping不通。这是因为服务的集群IP是一个虚拟IP，并且只有在与服务端口结合时才有意义。



### 8.5 扩容、缩容

*扩缩是通过改变 Deployment 中的副本数量来实现的。*

Kubernetes 还支持 Pods 的[自动缩放](https://kubernetes.io/docs/user-guide/horizontal-pod-autoscaling/)

在之前的模块中，我们创建了一个 [Deployment](https://kubernetes.io/docs/user-guide/deployments/)，然后通过 [Service](https://kubernetes.io/docs/user-guide/services/)让其可以开放访问。Deployment 仅为跑这个应用程序创建了一个 Pod。 当流量增加时，我们需要扩容应用程序满足用户需求。运行应用程序的多个实例需要在它们之间分配流量。服务 (Service)有一种负载均衡器类型，可以将网络流量均衡分配到外部可访问的 Pods 上。服务将会一直通过端点来监视 Pods 的运行，保证流量只分配到可用的 Pods 上。

```shell
#查看运行的ReplicaSet , DESIRED:需要的副本数  CURRENT当前运行的副本数
kubectl get rs
#设置4个副本数，通过改变replicas可以实现扩容和缩容
kubectl scale deployments/kubernetes-bootcamp --replicas=4
#查看service详情，可以看到对外暴露的端口，还有Endpoints中可以看到已经有4个副本了
#调用curl IP:NodePort 可以看到访问的pod是不一样的
kubectl describe services/kubernetes-bootcamp
```

### 8.6 滚动更新

*滚动更新允许通过使用新的实例逐步更新 Pod 实例从而实现 Deployments 更新，停机时间为零。*

*如果 Deployment 是公开的，则服务将仅在更新期间对可用的 pod 进行负载均衡。*

我们将应用程序扩展为运行多个实例。这是在不影响应用程序可用性的情况下执行更新的要求。默认情况下，更新期间不可用的 pod 的最大值和可以创建的新 pod 数都是 1。这两个选项都可以配置为（pod）数字或百分比。 在 Kubernetes 中，更新是经过版本控制的，任何 Deployment 更新都可以恢复到以前的（稳定）版本。

滚动更新允许以下操作：

- 将应用程序从一个环境提升到另一个环境（通过容器镜像更新）
- 回滚到以前的版本
- 持续集成和持续交付应用程序，无需停机

```shell
#通过set image命令滚动更新镜像到版本2
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
#检查守护进程的推出状态
kubectl rollout status deployments/kubernetes-bootcamp

#先设置一个不存在的镜像然后进行回滚
kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=gcr.io/google-samples/kubernetes-bootcamp:v10
# 查看状态
kubectl get deployments
kubectl get pods
kubectl describe pods
kubectl rollout undo deployments/kubernetes-bootcamp
```

```bash
kubectl rollout --help
Manage the rollout of a resource.
管理资源的部署。

 有效的资源类型包括：

  *  deployments
  *  daemonsets
  *  statefulsets

示例:
  # Rollback to the previous deployment
  # 回滚到上一个部署
  kubectl rollout undo deployment/abc

  # Check the rollout status of a daemonset
  # 检查守护进程的推出状态
  kubectl rollout status daemonset/foo

可用命令:
  history     显示 rollout 历史
  pause       标记提供的 resource 为中止状态
  restart     Restart a resource
              重新启动资源
  resume      继续一个停止的 resource
  status      显示 rollout 的状态
  undo        撤销上一次的 rollout

用法:
  kubectl rollout SUBCOMMAND [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).
```

### 8.7 使用 ConfigMap 来配置 Redis



### 8.8 kubectl 命令自动补全

**在linux上**

```shell
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
```

测试下，没问题后，我们对 /root/.bashrc 加2行代码 ，方便以后每次登录自动生效：

```shell
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
```

### 8.9 使用yaml创建pod

kubectl explain命令可以查看如何创建一个yaml文件每个对象有哪些属性

```shell
kubectl explain pods
kubectl explain pod.spec
```

yaml三大部分，编写的时候只需要编写metadata、spec 两部分

	metadata 包括名称、命名空间、标签和关于该容器的其他信息。
	spec 包含pod内容的实际说明，例如pod的容器、卷和其他数据。
	status 包含运行中的pod的当前信息，例如pod所处的条件、每个容器的描述状态，以及内部IP和其他基本信息
kubia-manual.yaml文件：

```yaml
apiVersion: v1 #描述文件遵守V1版本的Kubernetes API
kind: Pod #我们在描述一个pod
metadata:
	name: kubia-manual #pod的名称
spec:
	containers:
	- image: luksa/kubia #创建容器所使用的镜像
	  name: kubia #容器的名称
	  ports:
	  - containerPort: 8080 #应用监听的端口
	    protocol: TCP
```

使用 kubectl create -f kubia-manual.yaml 命令用于从yaml或JSON文件创建任何资源（不止pod）

加上 -n 可以指定创建到哪个namespace下面，也可以在metadata字段中添加一个namespace属性指定命名空间

kubia-manual-with-labels.yaml文件：

```yaml
apiVersion: v1 #描述文件遵守V1版本的Kubernetes API
kind: Pod #我们在描述一个pod
metadata:
	name: kubia-manual-v2 #pod的名称
	labels: #两个标签被附加到pod上
		creation_method: manual
		env: prod
spec:
	containers:
	- image: luksa/kubia #创建容器所使用的镜像
	  name: kubia #容器的名称
	  ports:
	  - containerPort: 8080 #应用监听的端口
	    protocol: TCP
```

### 8.10 标签

标签可以附加到任何Kubemetes对象上， 包括节点。

如果你的某些工作节点使用机械硬盘， 而其他节点使用固态硬盘， 那么你 可能想将 一 些pod调度到一 组节点 ，同时 将其他pod调度到另 一 组节点。 另外， 当需要将执行GPU密集型运算的pod调度到实际提供GPU加速的节点上时， 也需要pod调度。当运维团队向集群添加新节点时，他们将通过附加标签来对节点进行分类， 这些标签指定节点提供的硬件类型 ， 或者任何在调度pod时能提供便利的其他信息。

标签选择器根据资源的以下条件来选择资源：

​	• 包含（或不包含）使用特定键的标签

​	• 包含具有特定键和值的标签

​	• 包含具有特定键的标签， 但其值与我们指定的不同

kubia-gpu.yaml

```yaml
apiVersion: v1 #描述文件遵守V1版本的Kubernetes API
kind: Pod #我们在描述一个pod
metadata:
	name: kubia-manual-v2 #pod的名称
spec:
	nodeSelector: #节点选择器要求Kubeernetes只将pod部署到包含标签gpu=true的节点上
		gpu: "true"
	containers:
	- image: luksa/kubia #创建容器所使用的镜像
	  name: kubia #容器的名称
	  ports:
	  - containerPort: 8080 #应用监听的端口
	    protocol: TCP
```

kubectl annotate 命令 添加注解

kubectl get namespace 获取所有命名空间

namespace使我们能够将不属于--组的资源分到不重叠的组中。如果有多个用户或用户组正在使用同一个Kubernetes集群，并且它们都各自管理自己独特的资源集合，那么它们就应该分别使用各自的命名空间。这样一来，它们就不用特别担心无意中修改或删除其他用户的资源，也无须关心名称冲突。如前所述，命名空间为资源名称提供了一个作用域。

命名空间只包含字母、数字、横杠(-)。

yaml方式创建命名空间	kubectl create -f custom-namespace.yaml

```yaml
apiVersion: v1
kind: Namespace #这表示我们正在定义一个命名空间
metadata:
	name: custom-namespace #这是命名空间的名称
```

命令方式创建命名空间 kubectl create namespace custom-namespace

提示要想快速切换到不同的命名空间，可以设置以下别名:alias kcd=' kubectl config set-context $ (kubectl config current-context) --namespace'。 然后，可以使用kcd some-namespace在命名空间之间进行切换。

### 8.11 探测

> 如果你在容器中运行Java应用程序 ， 请确保使用HTTP GET存活探针，而不是启动全新NM以荻取存活信息的Exec探针。 任何基于NM或类似的应用程序也是如此 ， 它们的启动过程需要大量的计算资原。

Kubemetes 有以下三种探测容器的机制：
	HTTPGET 探针对容器的 IP 地址（你指定的端口和路径）执行 HTTP GET 请求。如果探测器收到响应，并且响应状态码不代表错误（换句话说，如果HTTP响应状态码是2xx或3xx), 则认为探测成功。如果服务器返回错误响应状态码或者根本没有响应，那么探测就被认为是失败的，容器将被重新启动。

kubia-liveness-probe.yaml

```yaml
apiVersion: v1 #描述文件遵守V1版本的Kubernetes API
kind: Pod #我们在描述一个pod
metadata:
	name: kubia-liveness #pod的名称
spec:
	containers:
	- image: luksa/kubia-unhealthy #创建容器所使用的镜像
	  name: kubia #容器的名称
	  livenessProbe: #一个http get存活探针
	  	httpGet:
	  		path: / #http请求的路径
	  		port: 8080 #探针连接的网络端口
	  	initialDelaySeconds: 15 #Kubernetes会在第一次探测前等待15秒。如果没有设置初始延迟，探针将在启动时立即开始探测容器， 这通常会导致探测失败， 因为应用程序还没准备好开始接收请求。 如果失败次数超过阅值，在应用程序能正确响应请求之前， 容器就会重启。
```

​	TCP套接字探针尝试与容器指定端口建立TCP连接。如果连接成功建立，则探测成功。否则，容器重新启动。

​	Exec探针在容器内执行任意命令，并检查命令的退出状态码。如果状态码是 0, 则探测成功。所有其他状态码都被认为失败。

### 8.12 ReplicationController

> pod 实例永远不会重新安置到另一个节点。 相反 ， ReplicationController 会创建一个全新的 pod 实例， 它与正在替换的实例无关。
>
> 模板中的pod标签显然必须和ReplicationController的标签选择器匹配，否则控制器将无休止地创建新的容器。
>
> 定义ReplicationController时不要指定pod选择器，让Kubemetes从pod模板中提取它。这样YAML更简短。
>
> 使用rc 作为 replicationController 的简写。
>
> 当使用 kubectl delete 删除 ReplicationController 时， 可以通过给命令增加 --cascade = false 选项来保持 pod 的运行。

ReplicationController有三个主要部分（如图4.3所示）：

​	label selector (标签选择器）， 用于确定ReplicationController作用域中有哪些pod

​	replica count (副本个数）， 指定应运行的pod 数量

​	pod template (pod模板）， 用于创建新的pod 副本

```yaml
apiVersion: v1 #描述文件遵守V1版本的Kubernetes API
kind: ReplicationController #我们在描述一个ReplicationController
metadata:
	name: kubia #名称
spec:
	replicas: 3 #pod 实例的目标数目
	selector:
		app: kubia #pod选择器决定了RC的操作对象
	template: #创建新pod所用的pod模版
		metadata:
			labels:
				app: kubia
		spec:
			containers:
			- name: kubia
			  image: luksa/kubia
			  ports:
			  - containerPort: 8080
```

### 8.13 ReplicaSet 

ReplicaSet 的行为与 ReplicationController 完全相同， 但 pod 选择器的表达能力更强。 虽然 ReplicationController 的标签选择器只允许包含某个标签的匹配 pod, 但ReplicaSet 的选择器还允许匹配缺少某个标签的 pod, 或包含特定标签名的 pod, 不管其值如何。举个例子， 单个 Repli氓ionController 无法将 pod 与标签 env=production和 env = devel 同时匹配。 它只能匹配带有 env = devel 标 签 的 pod 或带有env = devel 标签的 pod 。 但是 一 个 ReplicaSet 可以匹配两组 pod 并将它们视为 一 个大组。同样， 无论 ReplicationController 的值如何， ReplicationController 都无法仅基于标签名的存在来匹配 pod, 而 ReplicaSet 则可以。 例如， ReplicaSet 可匹配所有包含名为 env 的标签的 pod, 无论 ReplicaSet 的实际值是什么（可以理解为 env = *) 。

```yaml
apiVersion: apps/v1beta2 #描述文件遵守V1版本的Kubernetes API
kind: ReplicaSet #我们在描述一个ReplicationController
metadata:
	name: kubia #名称
spec:
	replicas: 3 #pod 实例的目标数目
	selector:
		matchLabels:
			app: kubia #这里使用了更简单的matchLabels选择器，这非常类似于ReplicationController的选择器
	template: #创建新pod所用的pod模版
		metadata:
			labels:
				app: kubia
		spec:
			containers:
			- name: kubia
			  image: luksa/kubia
			  ports:
			  - containerPort: 8080
```

matchExpressions选择器

```yaml
selector:
	matchExpressions:
	- key: app #此选择器要求该pod包含名为"app"的标签
	operator: In #In、NotIn、Exists、DoesNotExist
	values: #标签的值必须是“kubia”
		- kubia
```

### 8.14 DaemonSet

要在所有集群节点上运行 一 个pod, 需要创建 一 个DaemonSet对象， 这很像一个ReplicationController或ReplicaSet, 除了由DaemonSet 创建的pod, 已经有 一 个指定的目标节点并跳过Kubernetes调度程序。 它们不是随机分布在集群上的。

如果节点下线， DaemonSet不会在其他地方重新创建pod。 但是， 当将一 个新节点添加到集群中时， DaemonSet会立刻部署 一 个新的pod实例 。 如果有人无意中删除了一个pod,那么它也会重新创建- - 个新的pod。与ReplicaSet一样，DaemonSet从配置的pod模板创建pod。

DaemonSet将pod部署到集群中的所有节点上，除非指定这些pod只在部分节点上运行。这是通过pod模板中的nodeSelector属性指定的，这是DaemonSet定义的一部分(类似于ReplicaSet或ReplicationController中的pod模板)。

```yaml
apiVersion: apps/vlbeta2
kind: DaernonSet #DaemooSet在apps的API组中，版本是v1beta2
rnetadata:
	name: ssd-rnonitor
spec:
	selector:
		matchLabels:
			app: ssd-rnonitor
	template:
		metadata:
			labels:
				app: ssd-rnonitor
		spec:
			nodeSelector:
				disk: ssd # pod模板包含一个节点选择器，会选择有disk=ssd标签的节点
			containers:
				- name: main
				  image: luksa/ssd-monitor
```

### 8.15 Job

> Job可以配置为创建多个pod实例， 并以并行或串行方式运行它们 。通过在Job配置中设置 completions和parallelism属性来完成的 
>
> 通过在pod配置中设置 activeDeadlineSeconds属性，可以限制 pod的时间。如果 pod 运行时间超过此时间， 系统将尝试终止 pod, 并将 Job 标记为失败。通过指定 Job manifest 中的 spec.backoff巨m辽字段 ， 可以配置 Job在被标记为失败之前可以重试的次数。 如果你没有明确指定它， 则默认为6。

在发生节点故障时，该节点上由 Job管理的 pod将按照 ReplicaSet 的 pod 的方式，重新安排到其他节点。 如果进程本身异常退出（进程返回错误退出代码时）， 可以将 Job 配置为重新启动容器。一 旦任务完成， pod 就被认为处千完成状态。不重启容器

exporter.yaml

```yaml
apiVersion: batch/v1
kind: Job # Job属于batch API组 ，版本为v1
metadata:
	name: batch-job
spec:
	completions: 5 #将使此作业顺序运行5个pod。Job将 一 个接 一 个地运行五个pod。它最初创建 一 个pod, 当pod的容器运行完成时，它创建第二个pod, 以此类推，直到五个pod成功完成。
	parallelism: 2 #最多两个pod可以并行运行
	template:
		metadata:
			labels:
				app: batch-job #你没有指定pod选择器（它根据pod模版的标签创建）
		spec:
			restartPolicy: OnFailure #Job不能使用Always为默认的重新启动策略
			containers:
			- name: main
			  image: luksa/batch-job
```

通过pod配置的属性restartPolicy完成的， 默认为Always。 Jobpod不能使用默认策略， 因为它们不是要无限期地运行 。 因此， 需要明确地将重启策略设置为 OnFailure 或  Never。 此设置防止容器在完成任务时重新启动(pod
被Job管理时并不是这样的）。

### 8.16 CronJob

> Kubemetes中的 cron 任务通过创建 CronJob 资源进行配置。 运行任务的时间表以知名的 cron 格式指定， 所以如果你熟悉常规 cron 任务， 你将在几秒钟内了解Kubemetes 的 CronJob。

cornjob.yaml

```yaml
apiVersion: batch/vlbetal #API组是batch，版本是v1beta1
kind: CronJob
metadata:
	name: batch-job-every-fifteen-minutes
spec:
	schedule: "0,15,30,45  * * * *" #这项工作应该每天在每小时0、15、30和45分钟运行。
	startingDeadlineSeconds: 15 #pod最迟必须在预定时间后15秒开始运行工作。运行的时间应该是I0:30:00。如果因为任何原因10:30:15不启动，任务将不会运行，并将显示为Failed。
	jobTemplate: #此CronJob创建Job资源会用到的模板
		spec:
			template:
				metadata:
					labels:
						app: periodic-batch-job
				spec:
					restartPolicy: OnFailure
					containers:
					- name: main
					  image: luksa/batch-job
```

在正常 情况下，CronJob总是为计划中配置的每个执行创建 一 个 Job, 但可能 会同 时创建两个Job, 或者根本没有创建。为了解决第 一 个问题，你的任务应该是幕等的（多次而不是一 次运行不会得到不希望的结果）。对于第二个问题，请确保下一个任务运行完成本应该由上 一 次的（错过的） 运行完成的任何工作。

### 8.17 endpoint

> 服务并不是和 pod 直接相连的。相反，有一 种资源介于两者之间它就是 Endpoint 资源。
>
> 尽管在 spec 服务中定义了 pod 选择器，但在重定向传入连接时不会直接使用它。相反，选择器用于构建 IP 和端口列表，然后存储在 Endpoint 资源中。当客户端连接到服务时，服务代理选择这些 IP 和端口对中的 一 个，并将传入连接重定向到在该位置监听的服务器。
>
> kubectl get endpoints kubia

**endpoint手动配置**：如果创建了不包含 pod 选择器的服务， Kubemetes 将不会创建 Endpoint 资源（毕
竟，缺少选择器，将不会知道服务中包含哪些 pod) 。这样就需要创建 Endpoint 资源来指定该服务的 endpoint 列表。要使用手动配置 endpoint 的方式创建服务，需要创建服务和 Endpoint 资源。

external-service.yaml 不含pod选择器的服务

```yaml
apiVersion: v1
kind: Service
metadata:
	name: external-service #服务的名字必须和Endpoint对象的名字相匹配
spec: #服务中没有定义选择器
	ports:
	- port: 80
```

external-service-endpoints.yaml

```yaml
apiVersion:
kind: Endpoints
metadata:
	name: external-service #服务的名字必须和Endpoint对象的名字相匹配
subsets:
	- addresses: #服务将连接重定向到endpoint
		- ip: 11.11.11.11
		- ip: 22.22.22.22
		ports:
		- port: 80 #endpoint的目标端口
```

**为外部服务创建别名 ExternalName**，服 务 创 建 完成后，pod可以 通 过external-service.default . svc .cluster.local域 名 （甚至 是 external-service)连接到外部服务，而不是使用服务的实际FQDN。

external-service-externalname.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
	name: external-service 
spec:
	type: ExternalName #代码的type被设置成ExternalName
	externalName: someapi.somecompany.com #实际服务的完全限定域名
	ports:
	- port: 80
```

**将服务暴露给外部客户端**

有几种方式可以在外部访问服务：
	将服务的类型设置成NodePort-每个集群节点都会在节点上打开一个端口， 对于NodePort服务， 每个集群节点在节点本身（因此得名叫NodePort)上打开 一 个端口，并将在该端口上接收到的流量重定向到基础服务。该服务仅在内部集群 IP 和端口上才可访间， 但也可通过所有节点上的专用端口访问。

将类型设置为 NodePort并指定该服务应该绑定到的所有集群节点的节点端口。 指定端口不是强制性的。 如果忽略它，Kubemetes将选择 一 个随机端口。

kubia-svc-nodeport.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
	name:kubia-nodeport
spec:
	type: NodePort # 为NodePort设置服务类型
	ports:
	- port: 80 #服务集群Ip的端口号
	  targetPort: 8080 # 背后pod的目标端口号
	  nodePort: 30123 #通过集群节点的30123端口可以访问该服务
	selector:
		app: kubia
```

​	将服务的类型设置成LoadBalance, NodePort类型的 一 种扩展一 — 这使得服务可以通过 一 个专用的负载均衡器来访问， 这是由Kubernetes中正在运行的云基础设施提供的。 负载均衡器将流量重定向到跨所有节点的节点端口。客户端通过负载均衡器的 IP 连接到服务。

	在云提供商上运行的Kubernetes集群通常支持从云基础架构自动提供负载平衡器。所有需要做的就是设置服务的类型为Load Badancer而不是NodePort。负载均衡器拥有自己独一无二的可公开访问的 IP 地址，并将所有连接重定向到服务。可以通过负载均衡器的 IP 地址访问服务。如果Kubemetes在不支持Load Badancer服务的环境中运行，则不会调配负载平衡器，但该服务仍将表现得像一个NodePort服务。这是因为LoadBadancer服务是NodePo江服务的扩展。可以在支持Load Badancer服务的Google Kubemetes Engine上运行此示例。 Min灿be没有，至少在写作本书的时候。
kubia-svc-loadbalancer.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
	name: kubia-loadbalancer
sepc:
	type: LoadBalancer #该服务从Kubernetes集群的基础架构获取负载平衡器
	ports:
	- port: 80
	  targetPort: 8080
	selector:
		app: kubia
```

​	创建 一 个Ingress资源 ， 这是 一 个完全不同的机制， 通过一 个IP地址公开多个服务 —— 它运行在 HTTP 层（网络协议第 7 层）上， 因此可以提供比工作在第4层的服务更多的功能。 我们将在5.4节介绍Ingress资源。

> Ingress  (名词）一一 进入或进入的行为；进入的权利；进入的手段或地点；入口。
>
> 注意云供应商的Ingress控制器( 例如GKE)要求Ingress指向一个NodePort服务。但Kubernetes并没有这样的要求。
>
> 注意在云提供商的环境上运行时，地址可能需要一段时间才能显示，因为Ingress控制器在幕后调配负载均衡器。

一个重要的原因是每个 LoadBalancer 服务都需要自己的负载均衡器， 以及独有的公有 IP 地址， 而 Ingress 只需要 一 个公网 IP 就能为许多服务提供访问。 当客户端向 Ingress 发送 HTTP 请求时， Ingress 会根据请求的主机名和路径决定请求转发到的服务。Ingress 在网络栈 (HTTP) 的应用层操作， 并且可以提供 一 些服务不能实现的
功能， 诸如基于 cookie 的会话亲和性 (session affinity) 等功能。

kubia-ingress.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
	name: kubia
sepc:
	rules:
	- host: kubia.example.com #Ingress将域名kubia.example.com映射到你的服务
	  http:
	  	paths:
	  	- path: /kubia
	  	  backend: #将所有的请求发送到kubia-nodeport服务的80端口
	  	  	serviceName: kubia
	  	  	servicePort: 80
	  	 - path: /foo
	  	  backend: #将所有的请求发送到kubia-nodeport服务的80端口
	  	  	serviceName: foo
	  	  	servicePort: 80
	 - host: bar.example.com #Ingress将域名bar.example.com映射到你的服务
	  http:
	  	paths:
	  	- path: /
	  	  backend: #将所有的请求发送到kubia-nodeport服务的80端口
	  	  	serviceName: bar
	  	  	servicePort: 80
```

要通过htp://ubia.example.com访问服务，需要确保域名解析为Ingress控制器的IP。
获取Ingress的IP地址: kubectl get ingress 

**转发https请求**

当客户端创建到Ingress控制器的TLS连接时，控制器将终止TLS连接。客户端和控制器之间的通信是加密的，而控制器和后端pod之间的通信则不是。运行在pod.上的应用程序不需要支持TLS。例如，如果pod运行web服务器，则它只能接收HTTP通信，并让Ingress控制器负责处理与TLS相关的所有内容。要使控制器能够这样做，需要将证书和私钥附加到Ingress。这两个必需资源存储在称为Secret的Kubernetes资源中，然后在Ingress manifest中引用它。我们将在第7章中详细介绍Secret。现在，只需创建Secret,而不必太在意。

### 8.18 就绪探针

> 如何让pod准备就绪后才提供对外服务，使用就绪探针解决

像存活探针一样，就绪探针有三种类型:
	Exec探针，执行进程的地方。容器的状态由进程的退出状态代码确定。
	HTTP GET探针，向容器发送HTTP GET请求，通过响应的HTTP状态代码判断容器是否准备好。
	TCP socket探针，它打开一个TCP连接到容器的指定端口。如果连接已建立，则认为容器已准备就绪。

kubia-rc-readinessprobe.yaml

```yaml
#就绪探针将定期在容器内执行ls/var/ready命令。如果文件存在， 则ls 命令返回退出码 o, 否则返回非零的退出码。如果文件存在，则就绪探针将成功，否则，它会失败。
apiVersion: vl
kind: ReplicationController
...
spec:
	...
	template:
		...
		spec:
			containers:
			- name: kubia
			  image: luksa/kubia
			  readinessProbe: #pod中的每个容器都会有一个就绪探针
				exec:
				command:
				- ls
				- /var/ready
...
```

### 8.19 创建 headless 服务

> 将服务spec中的clusterIP字段设置为None会使服务成为headless服务，因为Kubernetes不会为其分配集群IP，客户端可通过该IP将其连接到支持它的pod.

kubia-svc-headless.yaml

```yaml
#使用 kubectl create 创建服务之后，可以 通过kubectl get和kubec口describe来查看服务，你会发现它没有集群IP, 并且它的后端 包含与pod选择器匹配的（部分） pod
apiVersion: vl
kind: Service
metadata:
	name: kubia-headless
spec:
	clusterIP: None #使服务成为headless
	ports:
	- port： 80
	  targetPort： 8080
	selector:
		app: kubia
```

**故障排查**

如果无法通过服务访问pod, 应该根据下面的列表进行排查：

​	首先， 确保从集群内连接到服务的集群IP, 而不是从外部。

​	不要通过ping服务IP 来判断服务是否可 访问（请记住， 服务的集群IP 是虚拟IP, 是无法ping通的）。

​	如果已经定义了就绪探针， 请确保 它返回成功；否则该pod不会成为服务的一部分 。

​	要确认某个容器是服务的 一 部分， 请使用kubectl ge七 endpoints 来检查相应的端点对象。

​	如 果尝试通过FQDN或其中一 部 分来访问服务（例如， myservice.mynamespace.svc.cluster.local 或 myservice.mynamespace), 但并不起作用， 请查看是否可以使用其集群IP而不是FQDN来访问服务 。

​	检查是否连接到服务公开的端口，而不是目标端口 。

​	尝试直接连接到podIP以确认pod正在接收正确端口上的 连接。

​	如果甚至无法通过pod的IP 访问应用， 请确保应用不是仅绑定 到本地主机。






## 9. 为集群增加service功能 - kube-proxy（工作节点）
9.1 简介

每台工作节点上都应该运行一个kube-proxy服务，它监听API server中service和endpoint的变化情况，并通过iptables等来为服务配置负载均衡，是让我们的服务在集群外可以被访问到的重要方式。
#### 9.2 部署
**通过系统服务方式部署：**

```bash
#确保工作目录存在
$ mkdir -p /var/lib/kube-proxy
#复制kube-proxy服务配置文件
$ cp target/worker-node/kube-proxy.service /lib/systemd/system/
#复制kube-proxy依赖的配置文件
$ cp target/worker-node/kube-proxy.kubeconfig /etc/kubernetes/

$ systemctl enable kube-proxy.service
$ service kube-proxy start
$ journalctl -f -u kube-proxy
```
##### 9.2.1 启动报错解决方案

```bash
Failed to execute iptables-restore: exit status 1 (iptables-restore: invalid option -- '5'
10月 24 15:50:38 server03 kube-proxy[16068]: Try `iptables-restore -h' for more information.
```

**解决方案为:**

```shell
# 问题一：kubectl get pods -o wide  查看deployments，显示pods 但是没有集群IP？
如下：
NAME                                 READY     STATUS    RESTARTS   AGE       IP              NODE
kubernates-bootcamp-7fddcf7f-b9s7l   1/1       Running   14         8h        <none>   192.168.2.108

如果创建pods、deploy，会反复出现CrashLoopBackOff 、Running等状态。 出现这种情况是因为  kube-proxy没配置好。参考下面。

问题二： kube-proxy  启动报错 Failed to execute iptables-restore: exit status 1？
这个问题已有相关文章阐述，但解决方式是 yum remove iptables ,这种方式太暴力，依赖iptables的相关组件太多（包括Docker），执行上述命令后，都会被卸载掉（可以参考：https://cloud.tencent.com/developer/article/1362616）。经多次测试，非常不可取。

本人使用rpm命令方式回滚iptables版本，仅重装iptables和iptables-services即可，缺少的包可以去 ，具体操作如下：

更换iptables的版本号降低到iptables-1.4.21-24.1.el7_5.x86，去http://rpm.pbone.net/ 下载下面两个包：
# 搜索到的下载地址 https://buildlogs.centos.org/c7.1804.u.x86_64/iptables/20180516080607/1.4.21-24.1.el7_5.x86_64/

iptables-1.4.21-24.1.el7_5.x86_64.rpm

iptables-services-1.4.21-24.1.el7_5.x86_64.rpm

然后非依赖移除iptables与iptables-services，命令参考如下：

rpm -e --nodeps  iptables-1.4.21-33.el7.x86_64

rpm -e --nodeps iptables-services-1.4.21-33.el7.x86_64

再安装上述下载的两个新包：

rpm -ivh iptables-1.4.21-24.1.el7_5.x86_64.rpm

rpm -ivh iptables-services-1.4.21-24.1.el7_5.x86_64.rpm

(也可以将两个包放在同一目录下，执行 rpm -ivh iptables-*.rpm)

重启kube-proxy服务既可以。
```



#### 9.3 重点配置说明

**kube-proxy.service**
> [Unit]  
Description=Kubernetes Kube-Proxy Server
...  
[Service]  
\#工作目录  
WorkingDirectory=/var/lib/kube-proxy  
ExecStart=/home/michael/bin/kube-proxy \\  
\#监听地址  
  --bind-address=192.168.1.103 \\  
  \#依赖的配置文件，描述了kube-proxy如何访问api-server  
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \\  
...

**kube-proxy.kubeconfig**
配置了kube-proxy如何访问api-server，内容与kubelet雷同，不再赘述。

#### 9.4 操练service
刚才我们在基础集群上演示了pod，deployments。下面就在刚才的基础上增加点service元素。具体内容见[《Docker+k8s微服务容器化实践》][1]。


## 10. 为集群增加dns功能 - kube-dns（app）
#### 10.1 简介
kube-dns为Kubernetes集群提供命名服务，主要用来解析集群服务名和Pod的hostname。目的是让pod可以通过名字访问到集群内服务。它通过添加A记录的方式实现名字和service的解析。普通的service会解析到service-ip。headless service会解析到pod列表。
#### 10.2 部署
**通过kubernetes应用的方式部署**
kube-dns.yaml文件基本与官方一致（除了镜像名不同外）。
里面配置了多个组件，之间使用”---“分隔
```bash
#到kubernetes-starter目录执行命令
$ kubectl create -f target/services/kube-dns.yaml
```
#### 10.3 重点配置说明
请直接参考配置文件中的注释。

#### 10.4 通过dns访问服务
这了主要演示增加kube-dns后，通过名字访问服务的原理和具体过程。演示启动dns服务和未启动dns服务的通过名字访问情况差别。
具体内容请看[《Docker+k8s微服务容器化实践》][1]吧~

[1]: https://coding.imooc.com/class/198.html
