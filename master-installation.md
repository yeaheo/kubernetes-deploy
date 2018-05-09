## kubernetes实践-部署master相关服务
- kubernetes master 节点包含的组件包括以下几个部分：
  ``` bash
  1、 kube-apiserver
  2、 kube-scheduler
  3、 kube-controller-manager
  ```
- 本次部署我们将三个组件安装在一台机器上，`kube-scheduler`、`kube-controller-manager` 和 `kube-apiserver` 三者的功能紧密相关；
- 同时只能有一个 `kube-scheduler`、`kube-controller-manager` 进程处于工作状态，如果运行多个，则需要通过选举产生一个 leader；
- master 节点上没有部署 `flannel` 网络插件，如果想要在 master 节点上也能访问 ClusterIP，请参考下一节部署ndoe节点中的配置Flanneld部分。

- **TLS证书文件**
- 以下pem证书文件我们在创建TLS证书和秘钥这一步中已经创建过了，`token.csv` 文件在创建 `kubeconfig` 文件的时候已经创建，具体证书如下：
  ``` bash
  [root@ceph-node1 ~]# ls /etc/kubernetes/ssl/
  admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem
  ```

### 部署 msater 节点相关服务
- 我们在安装 kubectl 工具的时候已经下载了 kubernetes server的软件包，可以直接使用已经下载的软件部署 master 节点。
- 但是，这里有两种安装方式，如下：
- **方式一：**
- 从 github release 页面 下载发布版 tarball，解压后再执行下载脚本，执行脚本后相关文件就会自动下载，但是由于网络的问题，我在测试的时候总是下载失败，所以采用了第二种安装方式。
- GitHub 下载地址：<https://github.com/kubernetes/kubernetes/releases>
  ``` bash
  wget https://github.com/kubernetes/kubernetes/releases/download/v1.9.6/kubernetes.tar.gz
  tar -xzvf kubernetes.tar.gz
  cd kubernetes
  ./cluster/get-kube-binaries.sh
  ```
- **方式二：**
- 从 CHANGELOG 页面 下载 client 或 server tarball 文件。
  > 需要注意的是：server 的 tarball kubernetes-server-linux-amd64.tar.gz 已经包含了 client(kubectl) 二进制文件，所以不用单独下载kubernetes-client-linux-amd64.tar.gz文件；
  ``` bash
  wget https://dl.k8s.io/v1.9.6/kubernetes-server-linux-amd64.tar.gz
  tar -xzvf kubernetes-server-linux-amd64.tar.gz
  cd kubernetes
  tar -xzvf  kubernetes-src.tar.gz
  ```
- 将二进制文件拷贝到指定路径:
  ``` bash
  cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/
  ```

#### 配置 kube-apiserver 服务
- **创建 kube-apiserver 的 systemd unit 文件**
- 我们需要自己创建 kube-apiserver 的服务启动文件 `/usr/lib/systemd/system/kube-apiserver.service`，具体内容如下:
- `cat /usr/lib/systemd/system/kube-apiserver.service`

  ``` service
  [Unit]
  Description=Kubernetes API Service
  Documentation=https://github.com/GoogleCloudPlatform/kubernetes
  After=network.target
  After=etcd.service

  [Service]
  EnvironmentFile=-/etc/kubernetes/config
  EnvironmentFile=-/etc/kubernetes/apiserver
  ExecStart=/usr/local/bin/kube-apiserver \
          $KUBE_LOGTOSTDERR \
          $KUBE_LOG_LEVEL \
          $KUBE_ETCD_SERVERS \
          $KUBE_API_ADDRESS \
          $KUBE_API_PORT \
          $KUBELET_PORT \
          $KUBE_ALLOW_PRIV \
          $KUBE_SERVICE_ADDRESSES \
          $KUBE_ADMISSION_CONTROL \
          $KUBE_API_ARGS
  Restart=on-failure
  Type=notify
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```
- 完整 `Systemd Unit` 文件参见 [kube-apiserver.service](https://github.com/yeaheo/kubernetes-manifests/blob/master/systemd/kube-apiserver.service)

- **创建 `/etc/kubernetes/config` 配置文件**
  ``` bash
  ###
  # kubernetes system config
  #
  # The following values are used to configure various aspects of all
  # kubernetes services, including
  #
  #   kube-apiserver.service
  #   kube-controller-manager.service
  #   kube-scheduler.service
  #   kubelet.service
  #   kube-proxy.service
  # logging to stderr means we get it in the systemd journal
  KUBE_LOGTOSTDERR="--logtostderr=true"
  
  # journal message level, 0 is debug
  KUBE_LOG_LEVEL="--v=0"
  
  # Should this cluster be allowed to run privileged docker containers
  KUBE_ALLOW_PRIV="--allow-privileged=true"
  
  # How the controller-manager, scheduler, and proxy find the apiserver
  KUBE_MASTER="--master=http://192.168.8.66:8080"
  ```
- 完整全局配置文件，参见 [config](https://github.com/yeaheo/kubernetes-manifests/blob/master/config/config)

  > 该配置文件同时被kube-apiserver、kube-controller-manager、kube-scheduler、kubelet、kube-proxy使用。所以我们需要将该文件复制到node所在机器上。

- **创建 `/etc/kubernetes/apiserver` 配置文件**
  ``` bash
  ###
  ## kubernetes system config
  ##
  ## The following values are used to configure the kube-apiserver
  ##
  #
  ## The address on the local server to listen to.
  #KUBE_API_ADDRESS="--insecure-bind-address=sz-pg-oam-docker-test-001.tendcloud.com"
  KUBE_API_ADDRESS="--advertise-address=192.168.8.66 --bind-address=192.168.8.66 --insecure-bind-address=192.168.8.66"
  
  #
  ## The port on the local server to listen on.
  #KUBE_API_PORT="--port=8080"
  #
  ## Port minions listen on
  #KUBELET_PORT="--kubelet-port=10250"
  #
  ## Comma separated list of nodes in the etcd cluster
  KUBE_ETCD_SERVERS="--etcd-servers=https://192.168.8.66:2379,https://192.168.8.67:2379,https://192.168.8.68:2379"
  
  #
  ## Address range to use for services
  KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
  #
  ## default admission control policies
  KUBE_ADMISSION_CONTROL="--admission-control=DefaultStorageClass,NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota"
  
  #--enable-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv 
  ## Add your own!
  KUBE_API_ARGS="--authorization-mode=Node,RBAC  --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true  --enable-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h"
  ```
- **参数说明**
- `--enable-bootstrap-token-auth` 该参数在 `v1.9` 版本已经成为正式的启动参数，但是在 `v1.6` 版本及以前的版本该参数为 `--experimental-bootstrap-token-auth`，`v1.7` 版本同 `v1.6` 版本， `v1.8` 版本同 `v1.9` 版本；
- 如果中途修改过 `--service-cluster-ip-range` 地址，则必须将 default 命名空间的 kubernetes 的 service 给删除，使用命令 `kubectl delete service kubernetes`，然后系统会自动用新的ip重建这个service。
- `--authorization-mode=Node,RBAC` 指定在安全端口使用 RBAC 授权模式，拒绝未通过授权的请求。 但是在 `v1.6` 版本中没有增加 `Node` 的授权模式。在 `v1.9` 版本中增加了 Node 授权的模式，否则将无法注册node;
- kube-scheduler、kube-controller-manager 一般和 kube-apiserver 部署在同一台机器上，它们使用非安全端口和 kube-apiserver 通信;
- kubelet、kube-proxy、kubectl 部署在其它 Node 节点上，如果通过安全端口访问 kube-apiserver，则必须先通过 TLS 证书认证，再通过 RBAC 授权；
- kube-proxy、kubectl 通过在使用的证书里指定相关的 User、Group 来达到通过 RBAC 授权的目的；
- 如果使用了 kubelet TLS Boostrap 机制，则不能再指定 `--kubelet-certificate-authority`、`--kubelet-client-certificate` 和 `--kubelet-client-key` 选项，否则后续 kube-apiserver 校验 kubelet 证书时出现 "x509: certificate signed by unknown authority" 错误；
- `--admission-control` 值必须包含 `ServiceAccount`；
- `--bind-address` 不能为 `127.0.0.1`；
- `--runtime-config` 配置为 `rbac.authorization.k8s.io/v1beta1`，表示运行时的 `apiVersion`；
- `--service-cluster-ip-range` 指定 `Service Cluster IP` 地址段，该地址段不能路由可达；
- 缺省情况下 kubernetes 对象保存在 etcd `/registry` 路径下，可以通过 `--etcd-prefix` 参数进行调整；
- 如果需要开通 http 的无认证的接口，则可以增加以下两个参数：`--insecure-port=8080 ``--insecure-bind-address=127.0.0.1`但是在生产环境上不要绑定到非 127.0.0.1 的地址上。

- 完整 `apiserver` 配置文件参见 [apiserver](https://github.com/yeaheo/kubernetes-manifests/blob/master/config/apiserver)

- **启动 kube-apiserver 服务**
  ``` bash
  systemctl daemon-reload
  systemctl start kube-apiserver
  systemctl enable kube-apiserver
  systemctl status kube-apiserver
  ```

#### 配置 kube-controller-manager 服务
- **创建 kube-controller-manager 的 systemd unit 文件**
- 我们需要自己创建 kube-apiserver 的服务启动文件 `/usr/lib/systemd/system/kube-controller-manager.service`，具体内容如下:

- `cat /usr/lib/systemd/system/kube-controller-manager.service`

  ``` bash
  [Unit]
  Description=Kubernetes Controller Manager
  Documentation=https://github.com/GoogleCloudPlatform/kubernetes
  
  [Service]
  EnvironmentFile=-/etc/kubernetes/config
  EnvironmentFile=-/etc/kubernetes/controller-manager
  ExecStart=/usr/local/bin/kube-controller-manager \
          $KUBE_LOGTOSTDERR \
          $KUBE_LOG_LEVEL \
          $KUBE_MASTER \
          $KUBE_CONTROLLER_MANAGER_ARGS
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```
- 完整 `Systemd Unit` 文件参见 [kube-controller-manager.service](https://github.com/yeaheo/kubernetes-manifests/blob/master/systemd/kube-controller-manager.service)

- **创建 `/etc/kubernetes/controller-manager` 配置文件**
  ``` bash
  ###
  # The following values are used to configure the kubernetes controller-manager
  
  # defaults from config and apiserver should be adequate
  
  # Add your own!
  KUBE_CONTROLLER_MANAGER_ARGS="--address=127.0.0.1 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true"
  ```

- **参数说明**
- `--service-cluster-ip-range` 参数指定 Cluster 中 Service 的CIDR范围，该网络在各 Node 间必须路由不可达，必须和 kube-apiserver 中的参数一致；
- `--cluster-signing-*` 指定的证书和私钥文件用来签名为 TLS BootStrap 创建的证书和私钥;
- `--root-ca-file` 用来对 kube-apiserver 证书进行校验，指定该参数后，才会在Pod 容器的 ServiceAccount 中放置该 CA 证书文件；
- `--address` 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 ontroller-manager 在同一台机器，否则会报错：“scheduler Unhealthy Get http://127.0.0.1:10251/healthz: "dial tcp 127.0.0.1:10251: getsockopt: connection refused "

- 完整 `controller-manager` 配置文件参见 [controller-manager](https://github.com/yeaheo/kubernetes-manifests/blob/master/config/controller-manager)

- **启动 kube-controller-manager 服务**
  ``` bash
  systemctl daemon-reload
  systemctl enable kube-controller-manager
  systemctl start kube-controller-manager
  systemctl status kube-controller-manager
  ```

#### 配置和启动 kube-scheduler 服务
- **创建 kube-scheduler 的 Systemd Unit 文件**
- 我们需要自己创建 kube-scheduler 的服务启动文件 `/usr/lib/systemd/system/kube-scheduler.service`，具体内容如下:
- `cat /usr/lib/systemd/system/kube-scheduler.service`
  
  ``` bash
  [Unit]
  Description=Kubernetes Scheduler Plugin
  Documentation=https://github.com/GoogleCloudPlatform/kubernetes
  
  [Service]
  EnvironmentFile=-/etc/kubernetes/config
  EnvironmentFile=-/etc/kubernetes/scheduler
  ExecStart=/usr/local/bin/kube-scheduler \
              $KUBE_LOGTOSTDERR \
              $KUBE_LOG_LEVEL \
              $KUBE_MASTER \
              $KUBE_SCHEDULER_ARGS
  Restart=on-failure
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```
- 完整 Systemd Unit 文件参见 [kube-scheduler.service](https://github.com/yeaheo/kubernetes-manifests/blob/master/systemd/kube-scheduler.service)

- **创建 `/etc/kubernetes/scheduler` 文件**
  
  ``` bash
  ###
  # kubernetes scheduler config
  
  # default config should be adequate
  
  # Add your own!
  KUBE_SCHEDULER_ARGS="--leader-elect=true --address=127.0.0.1"
  ```
- **参数说明**
- `--address` 值必须为 127.0.0.1，因为当前 kube-apiserver 期望 scheduler 和 controller-manager 在同一台机器；
- 完整 `scheduler` 配置文件参见 [scheduler](https://github.com/yeaheo/kubernetes-manifests/blob/master/config/scheduler)

- **启动 kube-scheduler 服务**
  ``` bash
  systemctl daemon-reload
  systemctl start kube-scheduler
  systemctl enable kube-scheduler
  systemctl status kube-scheduler
  ```

#### 验证 master 节点相关功能是否正常
- master 节点上的服务安装完成后，我们需要进一步验证其功能是否能正藏工作，具体参考如下：
  ``` bash
  [root@k8s-master system]# kubectl get componentstatus 
  NAME                 STATUS    MESSAGE              ERROR
  scheduler            Healthy   ok                   
  controller-manager   Healthy   ok                   
  etcd-2               Healthy   {"health": "true"}   
  etcd-1               Healthy   {"health": "true"}   
  etcd-0               Healthy   {"health": "true"}
  ```
- 从上面可以知道，我们部署的 master 节点基本服务都可以正常工作。  


