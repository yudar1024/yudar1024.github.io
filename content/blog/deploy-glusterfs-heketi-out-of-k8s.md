+++
author = "陈 sir"
categories = ["kubernetes"]
tags = ["kubernetes"]
date = "2019-09-10"
description = "deploy glusterfs and heketi out of kubernetes"
featured = ""
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "kubernetes 使用独立 glusterfs + heketi集群"
type = "post"

+++



**主机列表**

| Hostname | 服务器IP      | 存储IP        | 用途           | 硬盘信息 | 容量G |
| -------- | ------------- | ------------- | -------------- | -------- | ----- |
| master1  | 192.168.10.50 | 192.168.10.50 | hekteti server | /dev/sdb | 300G  |
| node1    | 192.168.10.51 | 192.168.10.51 | glusterfs      | /dev/sdb | 300G  |
| node2    | 192.168.10.52 | 192.168.10.53 | glusterfs      | /dev/sdb | 300G  |
| node3    | 192.168.10.53 | 192.168.10.53 | glusterfs      | /dev/sdb | 300G  |



**所有主机的/etc/hosts均包含**

``` bash
192.168.10.50 master1
192.168.10.51 node1
192.168.10.52 node2
192.168.10.53 node3
```

配置所有服务器之间免密登录（可选）

在node1，node2，node3 安装  glusterfs-server glusterfs-fuse，并加载fuse 内核模块

``` bash
#!/bin/bash
yum install -y centos-release-gluster
yum install -y glusterfs-server glusterfs-fuse
systemctl start glusterd
systemctl enable glusterd
modprobe fuse
echo "modprobe -- fuse" >> /etc/sysconfig/modules/glusterfs.modules
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
```

master1 安装 heketi-server 与heketi-client, 并加载 dm_thin_pool 内核模块

``` bash
yum install heketi heketi-client -y
modprobe dm_thin_pool
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
```

在master1生成ssh key，让其安装的heketi-server可以免密登录到node1，node2，node3

``` bash
ssh-keygen -t rsa -q -f /var/lib/heketi/id_rsa -N ''
chown heketi:heketi /var/lib/heketi/id_rsa*
ssh-copy-id -i /var/lib/heketi/id_rsa root@node1
ssh-copy-id -i /var/lib/heketi/id_rsa root@node2
ssh-copy-id -i /var/lib/heketi/id_rsa root@node3
```

编写heketi.json 配置文件

``` json
{
  "_port_comment": "Heketi Server Port Number",
  "port": "8888",

	"_enable_tls_comment": "Enable TLS in Heketi Server",
	"enable_tls": false,

	"_cert_file_comment": "Path to a valid certificate file",
	"cert_file": "",

	"_key_file_comment": "Path to a valid private key file",
	"key_file": "",


  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": false,

  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "_key_comment": "Set the admin key in the next line",
      "key": "openstack"
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "_key_comment": "Set the user key in the next line",
      "key": "openstack"
    }
  },

  "_backup_db_to_kube_secret": "Backup the heketi database to a Kubernetes secret when running in Kubernetes. Default is off.",
  "backup_db_to_kube_secret": false,

  "_profiling": "Enable go/pprof profiling on the /debug/pprof endpoints.",
  "profiling": false,

  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/var/lib/heketi/id_rsa",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab",
      "pv_data_alignment": "256K",
      "vg_physicalextentsize": "4MB",
      "lv_chunksize": "256K",
      "backup_lvm_metadata": false,
      "_debug_umount_failures": "Optional: boolean to capture more details in case brick unmounting fails",
      "debug_umount_failures": true
    },

    "_kubeexec_comment": "Kubernetes configuration",
    "kubeexec": {
      "host" :"https://kubernetes.host:8443",
      "cert" : "/path/to/crt.file",
      "insecure": false,
      "user": "kubernetes username",
      "password": "password for kubernetes user",
      "namespace": "OpenShift project or Kubernetes namespace",
      "fstab": "Optional: Specify fstab file on node.  Default is /etc/fstab",
      "pv_data_alignment": "Optional: Specify PV data alignment size. Default is 256K",
      "vg_physicalextentsize": "Optional: Specify VG physical extent size. Default is 4MB",
      "lv_chunksize": "Optional: Specify LV chunksize. Default is 256K",
      "xfs_sw": "Optional: Specify number of data disks in the underlying RAID device.",
      "xfs_su": "Optional: Specifies a stripe unit or RAID chunk size.",
      "backup_lvm_metadata": false,
      "gluster_cli_timeout": "Optional: Timeout, in seconds, passed to the gluster cli invocations",
      "_debug_umount_failures": "Optional: boolean to capture more details in case brick unmounting fails",
      "debug_umount_failures": true
    },

    "_db_comment": "Database file name",
    "db": "/var/lib/heketi/heketi.db",

     "_refresh_time_monitor_gluster_nodes": "Refresh time in seconds to monitor Gluster nodes",
    "refresh_time_monitor_gluster_nodes": 120,

    "_start_time_monitor_gluster_nodes": "Start time in seconds to monitor Gluster nodes when the heketi comes up",
    "start_time_monitor_gluster_nodes": 10,

    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
    "loglevel" : "debug",

    "_auto_create_block_hosting_volume": "Creates Block Hosting volumes automatically if not found or exsisting volume exhausted",
    "auto_create_block_hosting_volume": true,

    "_block_hosting_volume_size": "New block hosting volume will be created in size mentioned, This is considered only if auto-create is enabled.",
    "block_hosting_volume_size": 500,

    "_block_hosting_volume_options": "New block hosting volume will be created with the following set of options. Removing the group gluster-block option is NOT recommended. Additional options can be added next to it separated by a comma.",
    "block_hosting_volume_options": "group gluster-block",

    "_pre_request_volume_options": "Volume options that will be applied for all volumes created. Can be overridden by volume options in volume create request.",
    "pre_request_volume_options": "",

    "_post_request_volume_options": "Volume options that will be applied for all volumes created. To be used to override volume options in volume create request.",
    "post_request_volume_options": ""
  }
}

```

编写 topology.json 集群配置文件, 注意`storage` 这个key，如果是部署在K8S集群中的情况下，不需要。

``` json
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "node1"
              ],
              "storage": [
                "192.168.10.51"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node2"
              ],
              "storage": [
                "192.168.10.52"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "node3"
              ],
              "storage": [
                "192.168.10.53"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
```

将 heketi.json， topology.json 移动到/etc/heketi/ 目录下

```  bash 
mv heketi.json /etc/heketi/
mv topology.json /etc/heketi/
```

启动 heketi, 并通过heketi 读取topology.json 文件创建glusterfs 集群。 注意不要通过gluster probe 命令去创建，heketi 会自动创建集群。

``` bash
[root@master1 ~]# systemctl start heketi
[root@master1 ~]# systemctl enable heketi
[root@master1 ~]# export HEKETI_CLI_SERVER=http://192.168.10.50:8888
[root@master1 ~]# heketi-cli topology load --json=/etc/heketi/topology.json
[root@master1 ~]# heketi-cli cluster list
Clusters:
Id:9c483024cc203cde18f4c822fd1c562c [file][block]

```

## 验证

创建 storageclass

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: http://192.168.10.50:8888
  clusterid: 9c483024cc203cde18f4c822fd1c562c
  restauthenabled: "false"
  # restuser: "admin" # 此处需要与heketi 实际配置匹配，当restauthenabled 为false，需要被删除或注释掉
  # secretNamespace: "default"
  # secretName: "heketi-secret"
  #如果使用 secret 可以用如下命令新建对应的secret 
  # kubectl create secret generic heketi-secret \
  # --type="kubernetes.io/glusterfs" --from-literal=key='openstack' \
  # --namespace=default
  # 对应的yaml为 https://github.com/kubernetes/examples/blob/master/staging/persistent-volume-provisioning/glusterfs/glusterfs-secret.yaml
  # restuserkey: "openstack" # 此处需要与heketi 实际配置匹配，当restauthenabled 为false，需要被删除或注释掉
  gidMin: "40000" #存储类的 GID 范围的最小值和最大值。此范围内的唯一值 （GID）将用于动态预配卷。这些是可选值。如果未指定，则卷将预配值介于 2000-2147483647 之间，该值分别默认为 gidMin 和 gidMax。
  gidMax: "50000"
  # volumetype: "replicate:3" #Replica volume
  # volumetype: disperse:4:2  #Disperse/EC volume
  # volumetype: none  #Distribute volume
```

创建 pvc

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glusterfs-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: glusterfs-storage  #storage-class的名字
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

创建pod 使用pvc

nginx-test.yaml

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  labels:
    name: nginx-test
spec:
  containers:
  - name: nginx-test
    image: nginx
    ports:
    - name: web
      containerPort: 80
    volumeMounts:
    - name: gluster-vol1
      mountPath: /usr/share/nginx/html
  volumes:
    - name: gluster-vol1
      persistentVolumeClaim:
        claimName: glusterfs-claim
```

``` shell
kubectl apply -f nginx-test.yaml
# 在nginx 中添加内容，然后访问
kubectl exec -it nginx-test bash
cd /usr/share/nginx/html
echo "hellow gluster fs " >> index.html
ls
exit
kubectl forward nginx-test 1880:80
curl localhost:1880
```

## 安装失败问题

### 裸磁盘创建 device 失败
解决方法

 清除 vg
``` shell
[root@master1 deploy]# dmsetup ls
vg_d2ebbc4fc88ce69037db2da2b0a533f7-brick_90a958336d0e5b54b6f5643487805938      (253:6)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938-tpool   (253:4)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tdata   (253:3)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tmeta   (253:2)
vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938 (253:5)
# 注意顺序不能错，必须先brick 然后，不带任何字符的，然后 tpool ，tdata tmeta
$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-brick_90a958336d0e5b54b6f5643487805938

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938-tpool

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tdata

$dmsetup remove vg_d2ebbc4fc88ce69037db2da2b0a533f7-tp_90a958336d0e5b54b6f5643487805938_tmeta

```

 删除 gluster 相关文件
``` shell
rm -rf /var/lib/glusterd
```


 重置裸盘
``` shell
$dd if=/dev/zero of=/dev/sdb bs=1k count=1
blockdev --rereadpt /dev/sdb
```
5 重启

## pvc一直pending

此时有两种可能

1. 网络不通
2. storageclass 定义出现问题，或者heketi创建的集群出现问题

#### 第一种情况，

检查网络是否畅通，进入一个pod 去 telnet 或者 ping heketi server的ip， 如果能平通，进一步访问 

``` bash
curl http://<heketi-server-ip>:<heketi-server-port>/hello
```

如何能拿到返回，说明网络是通的。

#### 第二种情况

一般是heketi 初始化集群出现问题，需要删除heketi 的db，然后重新topology load 初始化集群，这个情况非常麻烦，甚至可能需要删除原先存在的volume。通过lsblk 命令查看gluster集群node的块存储情况，理论上应该是一样的，如果不一样，恭喜，需要重新初始化所有的glusterd systemd服务。

电信的错误日志为

``` bash
Warning  ProvisioningFailed  19s   persistentvolume-controller  Failed to provision volume with StorageClass "glusterfs-storage": failed to create volume: failed to update endpoint &Endpoints{ObjectMeta:k8s_io_apimachinery_pkg_apis_meta_v1.ObjectMeta{Name:glusterfs-dynamic-4b466ace-efba-48c7-b311-e3ff2a860ef0,GenerateName:,Namespace:default,SelfLink:,UID:,ResourceVersion:,Generation:0,CreationTimestamp:0001-01-01 00:00:00 +0000 UTC,DeletionTimestamp:<nil>,DeletionGracePeriodSeconds:nil,Labels:map[string]string{gluster.kubernetes.io/provisioned-for-pvc: glusterfs-claim,},Annotations:map[string]string{},OwnerReferences:[],Finalizers:[],ClusterName:,Initializers:nil,ManagedFields:[],},Subsets:[{[{node2  <nil> nil} {node3  <nil> nil} {node1  <nil> nil}] [] [{ 1 TCP}]}],}: <nil>
```

出现这个日志，可以去所有glusterfs 的node集群节点，通过lsblk 命令查看，回发先很多不一致的问题。

**解决方案**：

裸磁盘创建 device 失败的命令在各个node节点执行一遍。

先卸载整个集群

在node1,2,3 节点删除glusterfs-server

``` bash
systemctl stop glusterd
systemctl disable glusterd
yum erase glusterfs-server
rm -rf /etc/glusterfs
rm -rf /var/lib/glusterd
```

在master 节点删除 heketi数据文件

``` bash
rm /var/lib/heketi/heketi.db
```

重新初始化集群

在node1,2,3 节点重装glusterfs-server

``` bash
yum install -y glusterfs-server
systemctl start glusterd
systemctl enable glusterd
```

在master 节点重新初始化heketi topology

``` bash
heketi-cli topology load --json=/etc/heketi/topology.json
```



