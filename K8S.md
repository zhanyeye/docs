#### 架构

Kubernetes 集群里的节点分为 Master 和 Node 两种，其中 Master 负责管理集群，Node 负责运行应用容器。

![此处输入图片的描述](https://doc.shiyanlou.com/document-uid606277labid5982timestamp1531471340717.png)

Master 上运行的核心组件如下：

- API Server 是操作资源的唯一入口，提供认证、授权、访问控制、API 注册和发现等功能
- Scheduler 资源调度，按照预定的调度策略将 Pod 调度到相应的节点上
- Controller Manager 维护集群状态，比如故障检测、自动扩展、滚动更新等
- Etcd 保存集群状态

Node 上运行的核心组件如下：

- Docker 容器引擎，负责镜像管理以及运行容器，也可使用其它容器运行时（Container Runtime）
- Kubelet 管理容器的生命周期，同时也负责存储卷和网络的管理
- Kube-proxy 通过维护主机网络规则和连接转发来支持集群里的服务实现和负载均衡

除了核心组件，还有一些推荐的 Add-ons：

- Kube-dns 为集群内部提供 DNS 服务
- Ingress Controller 为服务提供外网入口
- Heapster 资源监控
- Fluentd 日志采集
- Dashboard 图形管理界面
- Federation 管理多个集群



#### 核心概念

+ Pod：Pod 是 Kubernetes 中最重要的技术概念。Pod 类似于 Docker 中的容器，是 Kubernetes 中的最小运行和管理单元。与容器不同的是，Pod 里可以包含一到多个相互紧密关联的容器，这些容器共享计算、存储和网络资源，因此可以通过进程间通信和文件共享这些简单高效的通信方式来组合成为服务。

+ 副本控制器（ReplicationController，RC）：RC 是 Kubernetes 最开始用来保证集群中 Pod 高可用的资源对象。它通过启动或删除 Pod 来保证运行中的 Pod 数量跟要求一致。RC 只适用于长期运行的服务型应用，它已经被更加强大的 RS 取代。

+ 副本集（ReplicaSet，RS）：RS 是 RC 的替代者，相比于 RC，它支持更多种的应用类型。RS 一般不单独使用，而是作为 Deployment 的期望状态来使用。

+ 部署（Deployment）：Deployment 用来描述应用的运行状态，包括运行多少个 Pod 副本，每个 Pod 里包含哪些容器，每个容器运行哪个镜像等。Deployment 创建完成之后，可以对其进行更新，比如修改镜像版本，扩容或缩减副本数量等。如果更新出现问题，还可以回滚到以前的版本。

+ 服务（Service）：通过 Deployment 我们能够完成应用部署，但如何访问应用提供的服务了？因为 Deployment 的 Pod 可能有多个，并且这些 Pod 所在的 Node 并不固定，因此没法使用固定的 IP 和端口去访问。Kubernetes 使用 Service 来解决此问题，一个 Service 对应一个应用，代表该应用提供的服务。每个 Service 有一个集群内部的虚拟 IP，客户端通过该 IP 来请求应用服务时，kube-proxy 会将请求转发给 Deployment 中的某个 Pod。当 Pod 位置发生变化时，kube-proxy 能够及时感知到。通过 kube-proxy 就解决了单个 Pod 服务的注册和发现问题，同时也实现了负载均衡。

+ 任务（Job）：Deployment 代表的是长期运行的应用服务，而短暂运行的应用（比如定时任务）就要用 Job 来表示。Job 有开始和结束，可以使用一个或多个 Pod 来执行。在多个 Pod 上运行时，运行成功可以配置为是其中一个完成还是全部都完成。

+ 后台支撑服务集（DaemonSet）：有时候希望在所有 Node 上都运行某个 Pod，比如网络路由、存储、日志、监控等服务，这个时候就可以使用 DaemonSet。

+ 有状态服务集（StatefulSet）：RS 中的 Pod 只能是无状态的，以便它们可以随时被销毁和重建。但有些时候不是这样，Pod 带有状态，比如数据库服务，在重建 Pod 的时候需要将之前的状态（也就是磁盘数据）恢复。使用 StatefulSet 可以达到此目的。StatefulSet 里的每个 Pod 都有名字，并且可以有顺序。当一个 Pod 被重建时，需要恢复之前的名字和相关资源（比如存储卷）。

+ 集群联邦（Federation）：部署在多个地区的 Kubernetes 集群可以以联邦的方式联合起来组成一个大的集群。每个对联邦的请求都会转发给联邦里的每个集群，每个集群都需要单独完成请求的操作。每个集群都是独立运行的，跟非联邦模式一模一样，它并不知晓自己属于某个联邦。这样就使得 Kubernetes 不用为引入联邦机制而对代码作任何改动。

+ 存储卷（Volume）：Volume 是存储的抽象，Kubernetes 中的存储卷跟 Docker 中的类似，只不过 Docker 中存储卷的作用范围是单个容器，而 Kubernetes 中是单个 Pod，被 Pod 中的多个容器共享。

+ 持久存储卷（Persistent Volume，PV）和持久存储卷声明（Persistent Volume Claim，PVC）就像 Node 提供计算资源，PV 提供了存储资源。PV 是对底层存储服务的抽象，其实现方式可以是本地磁盘，也可以是网络磁盘。PVC 用来描述 Pod 对存储资源的需求，它需要绑定到某个 PV。PV 和 PVC 是一对一关系，而 PV 和 Pod 是多对多关系，单个 PV 可以被多个 Pod 共享，且单个 Pod 可以绑定多个 PV。

+ 节点（Node）Node 负责提供计算资源，Pod 运行在 Node 之上。Node 可以是物理机也可以是虚拟机，甚至可以是容器（比如在单机上搭建 Kubernetes 测试集群的时候）。每个 Node 上都会运行一个 kubelet，以便 Master 可以管理这些节点。

+ 密钥对象（Secret）：Secret 对象用来存放密码、CA 证书等敏感信息，这些信息不适合直接用明文写在 Kubernetes 的对象配置文件里。Secret 对象可以由管理员预先创建好，然后在对象配置文件里通过名称来引用。这样可以降低敏感信息暴露的风险，也便于统一管理。
+ 用户帐号（User Account）和服务帐号（Service Account）：用户帐号为人提供身份标识，而服务帐号为 Kubernetes 集群中的 Pod 提供身份标识。用户帐号与命名空间无关，是跨命名空间的，而服务帐号属于某一个命名空间。
+ 命名空间（Namespace）：命名空间为同一个 Kubernetes 集群里的资源对象提供了虚拟的隔离空间，避免了命名冲突，比如在同一个集群里同时部署测试环境和生产环境服务。Kubernetes 里默认提供了两个命名空间，分别是 default 和 kube-system，前者是资源对象默认所属的空间，后者是 Kubernetes 自身资源对象所属的空间。只有集群管理员能够创建新的命名空间。
+  RBAC（Role-based Access Control，RBAC）访问授权：使用 RBAC，用户不再直接跟权限进行关联，而是通过角色。角色代表的一组权限，用户可以具备一种或多种角色，从而具有这些角色所包含的权限。如果角色权限有调整，那么所有具有该角色的用户权限自然而然就随之改变。



Kubernetes 中的对象

Kubernetes 里存在大量对象，基本上每个核心概念都对应有一个对象。这些对象除了在系统内部使用，也通过 API 开放给外部系统使用。
