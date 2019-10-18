+++
author = "陈 sir"
categories = ["kubernetes"]
tags = ["kubernetes"]
date = "2019-10-16"
description = "deploy rabbitmq cluster in kubernetes"
featured = "main/title.png"
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "kubernetes 部署 rabbitmq 集群"
type = "post"

+++


# RABBITMQ 集群基础知识
## 基础概念

- **Producer**: 消息发送者

- RabbitMQ

  :

  - Vhost: 相当于分组，每个vhost下数据是隔离的
  - Exchange: 路由器，接收消息，本根据RoutingKey分发消息
    - headers：消息头类型 路由器，内部应用
    - direct：精准匹配类型 路由器
    - topic：主题匹配类型 路由器，支持正则 模糊匹配
    - fanout：广播类型 路由器，RoutingKey无效
  - RoutingKey: 路由规则
  - Queue: 队列，用于存储消息（消息的目的地）

- **Consumer**: 消息消费者

## 持久化

RabbitMQ持久化分为Exchange、Queue、Message

- Exchange 和 Queue 持久化 指持久化Exchange、Queue 元数据，持久化的是自身，服务宕机，Exchange 和 Queue 自身就没有了
- Message 持久化 顾名思义 把每一条消息体持久化，服务宕机，消息不丢失



## 镜像队列

rabbitmq的队列（queue）镜像，指master node 在接受到请求后，会同步到其他节点上，以此来保证高可用。在confirm模式下，具体过程如下

clientpublisher 发送消息--> master node接到消息--> master node 将消息持久化到磁盘 --> 将消息异步发送给其他节点-->master 将ack 返回给client publisher。

> 需要理解的点，1 rabbitmq的消息接口是异步的。 publisher 发送给 master 之后，在没收到ack之前仍然可以继续发送消息，这中间没有阻塞。 2 master 在像其他节点同步消息的过程是阻塞性的，在这个工程中，队列不能读写。

### 镜像队列设置方法

镜像队列的配置通过添加policy完成，policy添加的命令为：

``` shell
rabbitmqctl set_policy [-p Vhost] Name Pattern Definition [Priority]

-p Vhost： 可选参数，针对指定vhost下的queue进行设置
Name: policy的名称
Pattern: queue的匹配模式(正则表达式)
Definition：镜像定义，包括三个部分ha-mode, ha-params, ha-sync-mode
	ha-mode:指明镜像队列的模式，有效值为 all/exactly/nodes
		all：表示在集群中所有的节点上进行镜像
		exactly：表示在指定个数的节点上进行镜像，节点的个数由ha-params指定
		nodes：表示在指定的节点上进行镜像，节点名称通过ha-params指定
	ha-params：ha-mode模式需要用到的参数
	ha-sync-mode：进行队列中消息的同步方式，有效值为automatic和manual
priority：可选参数，policy的优先级
————————————————
版权声明：本文为CSDN博主「朱小厮」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/u013256816/article/details/71097186
```

例如已mirror 开头的对里，都进行镜像

``` shell
rabbitmqctl set_policy --priority 0 --apply-to queues mirror_queue "^queue_" '{"ha-mode":"exactly","ha-params":2,"ha-sync-mode":"automatic"}'

```





## 当master节点挂掉之后，集群是如何处理的

1. 运行时间最长的镜像被提升为主镜像，假设它最有可能与主镜像完全同步。如果没有与主服务器同步的镜像，仅存在于主服务器上的消息将丢失。因为原master上的消息都已经落盘，所以也不会完全丢失。

2. mirror认为之前所有的消费者都被突然断开连接。它需要重发所有已发送到消费者但正在等待消费者确认的消息。这可能是客户端发出确认的消息，例如，如果消费者的确认消息在到达节点master之前网络上丢失（断网），或者在从master发送到到镜像主机时丢失。在这两种情况下，新主服务器别无选择，只能重新发送所有它没有看到确认的消息。（要求消费者对信息具有幂等性）
3. 当队列故障转移的时候，如果有消息被取消，publisher将会收到通知。
4. 消费者需要注意，作为重建队列的代价，消费者可能会重复收到之前消费过的消息。
5. 当选择的镜像成为主镜像时，在此期间发布到镜像队列的消息不会丢失。发布到镜像队列的消息会被路由到新的master 节点，然后同步到所有的mirror 节点。如果主服务器挂掉，消息将继续发送到镜像，并在镜像升级为master服务器完成后添加到队列中。
6. 发布者如果使用confirm模式，即使master 挂掉，消息仍然会被后面mirror 节点确认（提升为master），在发布者的角度，confirm模式发布的消息，发布到集群和发布到单节点，没什么不同。
7. 当一个镜像节点加入，或重新加入镜像队列是，之前的队列内容全部丢失。
8. 当所有slave都处在(与master)未同步状态时，并且ha-promote-on-shutdown policy设置为when-syned(默认)时，如果master因为主动的原因停掉，比如是通过rabbitmqctl stop命令停止或者优雅关闭OS，那么slave不会接管master，也就是说此时镜像队列不可用；但是如果master因为被动原因停掉，比如VM 或者OS crash了，那么slave会接管master。这个配置项隐含的价值取向是优先保证消息可靠不丢失，放弃可用性。如果ha-promote-on- shutdown policy设置为alway，那么不论master因为何种原因停止，slave都会接管master，优先保证可用性。
9. 镜像队列中最后一个停止的节点会是master，启动顺序必须是master先起，如果slave先起，它会有30秒的等待时间，等待master启动， 然后加入cluster。当所有节点因故(断电等)同时离线时，每个节点都认为自己不是最后一个停止的节点。要恢复镜像队列，可以尝试在30秒之内同时启 动所有节点。
10. 对于镜像队列，客户端basic.publish操作会同步到所有节点；而其他操作则是通过master中转，再由master将操作作用于salve。 比如一个basic.get操作，假如客户端与slave建立了TCP连接，首先是slave将basic.get请求发送至master，由 master备好数据，返回至slave，投递给消费者。
11. 由10可知，当slave宕掉时，除了与slave相连的客户端连接全部断开之外，没有其他影响。当master宕掉时，会有以下连锁反应：1)与 master相连的客户端连接全部断开。2)选举最老的slave为master。若此时所有slave处于未同步状态，则未同步部分消息丢失。3)新的 master节点requeue所有unack消息，因为这个新节点无法区分这些unack消息是否已经到达客户端，亦或是ack消息丢失在到老 master的通路上，亦或是丢在老master组播ack消息到所有slave的通路上。所以处于消息可靠性的考虑，requeue所有unack的消 息。此时客户端可能受到重复消息。4)如果客户端连着slave，并且basic.consume消息时指定了x-cancel-on-ha- failover参数，那么客户端会收到一个Consumer Cancellation Notification通知，Java SDK中会回调Consumer接口的handleCancel()方法，故需覆盖此方法。如果不指定x-cancel-on-ha-failover参 数，那么消费者就无法感知master宕机，会一直等待下去。

## 流控制

rabbitmq 通过凭证算法来控制消息发送的频率，只有当发布者从所有的mirror 节点获取凭证(credit),后才能发布消息。如果mirror 节点故障无法发布凭证，发布者将无法发布消息，直到故障修复，或者集群其他检点认为故障节点已经脱离集群。

## 数据保障

ha-promote-on-failure 策略（policy key），如果设置为`when-synced` 那么数据与master 节点不同步的节点，讲无法被选择为新的master。 但是这样整个集群的高可用将会降低。取决于一致性与集群可用性的取舍。默认值为`always`



## 故障恢复

下面的镜像队列恢复才是本文重点：

- 前提：两个节点(A和B)组成一个镜像队列。
- 场景1：A先停，B后停。

该场景下B是master，只要先启动B，再启动A即可。或者先启动A，再在30秒之内启动B即可恢复镜像队列。

- 场景2: A, B同时停。

该场景可能是由掉电等原因造成，只需在30秒之内连续启动A和B即可恢复镜像队列。

- 场景3：A先停，B后停，且A无法恢复。

该场景是场景1的加强版，因为B是master，所以等B起来后，在B节点上调用rabbitmqctl forget_cluster_node A，解除与A的cluster关系，再将新的slave节点加入B即可重新恢复镜像队列。

- 场景4：A先停，B后停，且B无法恢复。

该场景是场景3的加强版，比较难处理，早在3.1.x时代之前貌似都没什么好的解决方法，可能是我不知道，但是现在已经有解决方法了，在3.4.2 版本亲测有效。因为B是master，所以直接启动A是不行的，当A无法启动时，也就没办法在A节点上调用rabbitmqctl forget_cluster_node B了。新版本中，forget_cluster_node支 持–offline参数，offline参数允许rabbitmqctl在离线节点上执行forget_cluster_node命令，迫使 RabbitMQ在未启动的slave节点中选择一个作为master。当在A节点执行rabbitmqctl forget_cluster_node –offline B时，RabbitMQ会mock一个节点代表A，执行forget_cluster_node命令将B剔出cluster，然后A就能正常启动了。最后 将新的slave节点加入A即可重新恢复镜像队列。

- 场景5: A先停，B后停，且A、B均无法恢复，但是能得到A或B的磁盘文件。

该场景是场景4的加强版，更加难处理。将A或B的数据库文件(默认在$RABBIT_HOME/var/lib目录中)拷贝至新节点C的目录下，再 将C的hostname改成A或B的hostname。如果拷过来的是A节点磁盘文件，按场景4处理方式；如果拷过来的是B节点磁盘文件，按场景3处理方 式。最后将新的slave节点加入C即可重新恢复镜像队列。

- 场景6：A先停，B后停，且A、B均无法恢复，且无法得到A或B的磁盘文件。

洗洗睡吧，该场景下已无法恢复A、B队列中的内容了。

[镜像队列](https://blog.csdn.net/u013256816/article/details/71097186)



# rabbitmq 部署

## 服务发现

rabbitmq 在K8S 集群中 通过`rabbitmq_peer_discovery_k8s` plugin 与apiserver 进行交付，获取各个服务的URL,RABBITMQ 在K8S 集群中必须是用 statefulset 和 headless service 进行匹配。



部署rabbitmq 到K8S 集群中

创建 namespace 用于测试

``` shell
kubectl create ns test-rabbitmq
```

创建用于rbac的serviceaccount 权限文件

``` yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: rabbitmq
  namespace: test-rabbitmq
# Required from EKS 1.13
# https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server
automountServiceAccountToken: true
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: endpoint-reader
  namespace: test-rabbitmq
rules:
- apiGroups: [""]
  resources: ["endpoints"]
  verbs: ["get"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: endpoint-reader
  namespace: test-rabbitmq
subjects:
- kind: ServiceAccount
  name: rabbitmq
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: endpoint-reader
```

statefulsets.yaml 部署文件

``` yaml
kind: Service
apiVersion: v1
metadata:
  namespace: test-rabbitmq
  name: rabbitmq
  labels:
    app: rabbitmq
    type: LoadBalancer
spec:
  type: NodePort
  ports:
   - name: http
     protocol: TCP
     port: 15672
     targetPort: 15672
     nodePort: 31672
   - name: amqp
     protocol: TCP
     port: 5672
     targetPort: 5672
     nodePort: 30672
  selector:
    app: rabbitmq
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq-config
  namespace: test-rabbitmq
data:
  enabled_plugins: |
      [rabbitmq_management,rabbitmq_peer_discovery_k8s].
  rabbitmq.conf: |
      ## Cluster formation. See https://www.rabbitmq.com/cluster-formation.html to learn more.
      cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
      cluster_formation.k8s.host = kubernetes.default.svc.cluster.local
      ## Should RabbitMQ node name be computed from the pod's hostname or IP address?
      ## IP addresses are not stable, so using [stable] hostnames is recommended when possible.
      ## Set to "hostname" to use pod hostnames.
      ## When this value is changed, so should the variable used to set the RABBITMQ_NODENAME
      ## environment variable.
      cluster_formation.k8s.address_type = hostname
      ## How often should node cleanup checks run?
      cluster_formation.node_cleanup.interval = 30
      ## Set to false if automatic removal of unknown/absent nodes
      ## is desired. This can be dangerous, see
      ##  * https://www.rabbitmq.com/cluster-formation.html#node-health-checks-and-cleanup
      ##  * https://groups.google.com/forum/#!msg/rabbitmq-users/wuOfzEywHXo/k8z_HWIkBgAJ
      cluster_formation.node_cleanup.only_log_warning = true
      cluster_partition_handling = autoheal
      ## See https://www.rabbitmq.com/ha.html#master-migration-data-locality
      queue_master_locator=min-masters
      ## This is just an example.
      ## This enables remote access for the default user with well known credentials.
      ## Consider deleting the default user and creating a separate user with a set of generated
      ## credentials instead.
      ## Learn more at https://www.rabbitmq.com/access-control.html#loopback-users
      loopback_users.guest = false
---
apiVersion: apps/v1
# See the Prerequisites section of https://www.rabbitmq.com/cluster-formation.html#peer-discovery-k8s.
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: test-rabbitmq
spec:
  selector:
    matchLabels:
      app: rabbitmq 
  serviceName: rabbitmq
  # Three nodes is the recommended minimum. Some features may require a majority of nodes
  # to be available.
  replicas: 3
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      serviceAccountName: rabbitmq
      terminationGracePeriodSeconds: 10
      nodeSelector:
        # Use Linux nodes in a mixed OS kubernetes cluster.
        # Learn more at https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#kubernetes-io-os
        kubernetes.io/os: linux
      containers:
      - name: rabbitmq-k8s
        image: rabbitmq:3.8
        volumeMounts:
          - name: config-volume
            mountPath: /etc/rabbitmq
        # Learn more about what ports various protocols use
        # at https://www.rabbitmq.com/networking.html#ports
        ports:
          - name: http
            protocol: TCP
            containerPort: 15672
          - name: amqp
            protocol: TCP
            containerPort: 5672
        livenessProbe:
          exec:
            # This is just an example. There is no "one true health check" but rather
            # several rabbitmq-diagnostics commands that can be combined to form increasingly comprehensive
            # and intrusive health checks.
            # Learn more at https://www.rabbitmq.com/monitoring.html#health-checks.
            #
            # Stage 2 check:
            command: ["rabbitmq-diagnostics", "status"]
          initialDelaySeconds: 60
          # See https://www.rabbitmq.com/monitoring.html for monitoring frequency recommendations.
          periodSeconds: 60
          timeoutSeconds: 15
        readinessProbe:
          exec:
            # This is just an example. There is no "one true health check" but rather
            # several rabbitmq-diagnostics commands that can be combined to form increasingly comprehensive
            # and intrusive health checks.
            # Learn more at https://www.rabbitmq.com/monitoring.html#health-checks.
            #
            # Stage 2 check:
            command: ["rabbitmq-diagnostics", "status"]
            # To use a stage 4 check:
            # command: ["rabbitmq-diagnostics", "check_port_connectivity"]
          initialDelaySeconds: 20
          periodSeconds: 60
          timeoutSeconds: 10
        imagePullPolicy: Always
        env:
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
          - name: RABBITMQ_USE_LONGNAME
            value: "true"
          # See a note on cluster_formation.k8s.address_type in the config file section
          - name: K8S_SERVICE_NAME
            value: rabbitmq
          - name: RABBITMQ_NODENAME
            value: rabbit@$(MY_POD_NAME).$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local
          - name: K8S_HOSTNAME_SUFFIX
            value: .$(K8S_SERVICE_NAME).$(MY_POD_NAMESPACE).svc.cluster.local
          - name: RABBITMQ_ERLANG_COOKIE
            value: "mycookie"
      volumes:
        - name: config-volume
          configMap:
            name: rabbitmq-config
            items:
            - key: rabbitmq.conf
              path: rabbitmq.conf
            - key: enabled_plugins
              path: enabled_plugins
```



[一些相关的配置文件](http://next.rabbitmq.com/cluster-formation.html#peer-discovery-k8s)

官方 配置案例

[传送门](https://github.com/rabbitmq/rabbitmq-peer-discovery-k8s/tree/master/examples/k8s_statefulsets)



