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



