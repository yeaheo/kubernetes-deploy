## kubernetes实践-部署 node 相关服务
- kubernetes node 节点包含如下组件：
  ``` bash
  Flanneld
  Docker:docker直接用yum安装即可
  kubelet
  kube-proxy
  注意：每台 node 上都需要安装 flannel，master 节点上可以不必安装。
  ```
- 我们在前面已经配置了 Flanneld 和 Docker 服务，具体参见 [配置 docker 及 flanneld 服务](./flannel-net-installation.md)
- 我们本次只安装 `kubelet` 和 `kube-proxy` 服务。
  
  > kubernets `v1.9` 相对于 kuberentes `v1.6` 集群，必须关闭swap，否则kubelet启动将失败。
  > 关闭 `swap` 只需修改 `/etc/fstab` 将，`swap` 系统注释掉。
  > 如果要临时关闭，可以用 `swapoff -a`， `-a` 表示关闭所有交换设备。

### 安装并配置 kubelet 服务

- kubelet 启动时向 `kube-apiserver` 发送 `TLS bootstrapping` 请求，需要先将 `bootstrap token`文件中的 `kubelet-bootstrap` 用户赋予 `system:node-bootstrapper cluster` 角色(role)， 然后 kubelet 才能有权限创建认证请求(certificate signing requests)：
  ``` bash
  cd /etc/kubernetes
  kubectl create clusterrolebinding kubelet-bootstrap \
    --clusterrole=system:node-bootstrapper \
    --user=kubelet-bootstrap
  ```
  > `--user=kubelet-bootstrap` 是在 `/etc/kubernetes/token.csv` 文件中指定的用户名，同时也写入了 `/etc/kubernetes/bootstrap.kubeconfig` 文件；
- 下载最新的 kubelet 和 kube-proxy 二进制文件,因为我们之前已经下载了 kubernetes 的 server 软件包，所以再贴一下，不再赘述：
  ``` bash
  wget https://dl.k8s.io/v1.9.6/kubernetes-server-linux-amd64.tar.gz
  tar -xzvf kubernetes-server-linux-amd64.tar.gz
  cd kubernetes
  tar -xzvf  kubernetes-src.tar.gz
  cp -r ./server/bin/{kube-proxy,kubelet} /usr/local/bin/
  ```
- **创建 kubelet 的 System Unit 配置文件**
- kubelet 的 service 配置文件: `/usr/lib/systemd/system/kubelet.service`
  ``` bash
  [Unit]
  Description=Kubernetes Kubelet Server
  Documentation=https://github.com/GoogleCloudPlatform/kubernetes
  After=docker.service
  Requires=docker.service
  
  [Service]
  WorkingDirectory=/var/lib/kubelet
  EnvironmentFile=-/etc/kubernetes/config
  EnvironmentFile=-/etc/kubernetes/kubelet
  ExecStart=/usr/local/bin/kubelet \
              $KUBE_LOGTOSTDERR \
              $KUBE_LOG_LEVEL \
              $KUBELET_API_SERVER \
              $KUBELET_ADDRESS \
              $KUBELET_PORT \
              $KUBELET_HOSTNAME \
              $KUBE_ALLOW_PRIV \
              $KUBELET_POD_INFRA_CONTAINER \
              $KUBELET_ARGS
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target
  ```
- kubelet 完整 Systemd Unit 文件参见 [kubelet.service](https://github.com/yeaheo/kubernetes-manifests/blob/master/systemd/kubelet.service)
  > 在启动 kubelet 之前，需要先手动创建 `/var/lib/kubelet` 目录。

- **创建 kubelet 的配置文件**
  > 本次安装的 kubernetes 的版本是 `v1.9.6` 版本，相对于较早版本 `v1.6`，有点变化：取消了 `KUBELET_API_SERVER` 的配置，而改用 kubeconfig 文件来定义 master 地址。 所以需要我们注释掉 `KUBELET_API_SERVER` 配置。

- 修改后的 kubelet 配置文件 `/etc/kubernetes/kubelet` 参数如下：
  ``` bash
  ###
  ## kubernetes kubelet (minion) config
  #
  ## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
  KUBELET_ADDRESS="--address=192.168.8.67"
  #
  ## The port for the info server to serve on
  #KUBELET_PORT="--port=10250"
  #
  ## You may leave this blank to use the actual hostname
  KUBELET_HOSTNAME="--hostname-override=k8s-node1"
  #
  ## location of the api-server
  #KUBELET_API_SERVER="--api-servers=http://192.168.8.66:8080"   v1.8+ 版本需要注释掉此参数
  #
  ## pod infrastructure container
  KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=192.168.8.69/library/pause-amd64:3.0"
  #
  ## Add your own!
  KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=10.254.0.2 --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig  --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false --cgroups-per-qos=false --enforce-node-allocatable="" "
  ```
- **参数说明**：
- `--address` 不能设置为 127.0.0.1，否则后续 Pods 访问 kubelet 的 API 接口时会失败，因为 Pods 访问的 127.0.0.1 指向自己而不是 kubelet；
- 如果设置了 `--hostname-override` 选项，则 `ube-proxy` 也需要设置该选项，否则会出现找不到 Node 的情况；
- `--cgroup-driver` 配置成 `systemd`，不要使用cgroup，否则在 CentOS 系统中 kubelet 将启动失败（保持docker和kubelet中的cgroup driver配置一致即可，不一定非使用systemd）。
- `--experimental-bootstrap-kubeconfig` 指向 `bootstrap kubeconfig` 文件，kubelet 使用该文件中的用户名和 token 向 kube-apiserver 发送 TLS Bootstrapping 请求；
- 管理员通过了 CSR 请求后，kubelet 自动在 `--cert-dir` 目录创建证书和私钥文件(kubelet-client.crt 和 kubelet-client.key)，然后写入 `--kubeconfig` 文件；
- 建议在 `--kubeconfig` 配置文件中指定 `kube-apiserver` 地址，如果未指定 `--api-servers` 选项，则必须指定 `--require-kubeconfig` 选项后才从配置文件中读取 kube-apiserver 的地址，否则 kubelet 启动后将找不到 kube-apiserver (日志中提示未找到 API Server），kubectl get nodes 不会返回对应的 Node 信息;
- `--cluster-dns` 指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，`--cluster-domain` 指定域名后缀，这两个参数同时指定后才会生效；
- `--cluster-domain` 指定 pod 启动时 `/etc/resolve.conf` 文件中的 search domain ，起初我们将其配置成了 cluster.local.，这样在解析 service 的 DNS 名称时是正常的，可是在解析 headless service 中的 FQDN pod name 的时候却错误，因此我们将其修改为 cluster.local，去掉嘴后面的 ”点号“ 就可以解决该问题，关于 kubernetes 中的域名/服务名称解析请参见我的另一篇文章。
- `--kubeconfig=/etc/kubernetes/kubelet.kubeconfig` 中指定的 `kubelet.kubeconfig` 文件在第一次启动 kubelet 之前并不存在，请看下文，当通过CSR请求后会自动生成 `kubelet.kubeconfig `文件，如果你的节点上已经生成了 `~/.kube/config` 文件，你可以将该文件拷贝到该路径下，并重命名为 `kubelet.kubeconfig`，所有 node 节点可以共用同一个 `kubelet.kubeconfig` 文件，这样新添加的节点就不需要再创建CSR请求就能自动添加到 kubernetes 集群中。同样，在任意能够访问到 kubernetes 集群的主机上使用 `kubectl --kubeconfig` 命令操作集群时，只要使用 `~/.kube/config` 文件就可以通过权限认证，因为这里面已经有认证信息并认为你是 admin 用户，对集群拥有所有权限。
- `KUBELET_POD_INFRA_CONTAINER` 是基础镜像容器，这里我用的是私有镜像仓库地址，大家部署的时候需要修改为自己的镜像。 pod-infrastructure 镜像是 Redhat 制作的，大小接近80M，下载比较耗时，其实该镜像并不运行什么具体进程，可以使用 Google 的 pause 镜像 `gcr.io/google_containers/pause-amd64:3.0`，这个镜像只有300多K，比较容易下载和配置。

- 完整 kubelet 配置文件参见 [kubelet](https://github.com/yeaheo/kubernetes-manifests/blob/master/config/kubelet)

  > 其他 node 节点上的 kubelet 配置文件对应的 IP 地址需要改为每台 node 节点的 IP 地址。
  > 主要需要修改 `KUBELET_ADDRESS` 和 `KUBELET_HOSTNAME` 参数。

- **启动 kubelet 服务**
- 我们修改完相关参数后就可以启动 kubelet 服务了。
  ``` bash
  systemctl daemon-reload
  systemctl start kubelet
  systemctl enable kubelet
  systemctl status kubelet
  ```
- **通过 kublet 的 TLS 证书请求**
- kubelet 首次启动时向 kube-apiserver 发送证书签名请求，必须通过后 kubernetes 系统才会将该 Node 加入到集群。
- 查看未授权的 CSR 请求:
  ``` bash
  $ kubectl get csr
  NAME        AGE       REQUESTOR           CONDITION
  csr-2b308   4m        kubelet-bootstrap   Pending
  $ kubectl get nodes
  No resources found.
  ```
- 通过 CSR 请求:
  ``` bash
  $ kubectl certificate approve csr-2b308
  certificatesigningrequest "csr-2b308" approved
  [root@k8s-master kubernetes]# kubectl get nodes
  NAME        STATUS    ROLES     AGE       VERSION
  k8s-node1   Ready     <none>    34d       v1.9.6
  ```
- 此时，自动生成了 kubelet kubeconfig 文件和公私钥：
  ``` bash
  [root@k8s-node1 ~]# ls -l /etc/kubernetes/kubelet.kubeconfig
  -rw------- 1 root root 2215 Apr  6 18:15 /etc/kubernetes/kubelet.kubeconfig
  [root@k8s-node1 ~]# ls -l /etc/kubernetes/ssl/kubelet*
  -rw-r--r-- 1 root root 1042 Apr  6 18:15 /etc/kubernetes/ssl/kubelet-client.crt
  -rw------- 1 root root  227 Apr  6 18:08 /etc/kubernetes/ssl/kubelet-client.key
  -rw-r--r-- 1 root root 1111 Apr  6 18:08 /etc/kubernetes/ssl/kubelet.crt
  -rw------- 1 root root 1679 Apr  6 18:08 /etc/kubernetes/ssl/kubelet.key
  ```
- 假如你更新 kubernetes 的证书，只要没有更新 token.csv，当重启 kubelet 后，该 node 就会自动加入到 kuberentes 集群中，而不会重新发送 certificaterequest，也不需要在 master 节点上执行 `kubectl certificate approve`操作。前提是不要删除 node 节点上的`/etc/kubernetes/ssl/kubelet*` 和 `/etc/kubernetes/kubelet.kubeconfig` 文件。否则kubelet启动时会提示找不到证书而失败。
  > 如果启动 kubelet 的时候见到证书相关的报错，有个 trick 可以解决这个问题，可以将 master 节点上的 `~/.kube/config` 文件（该文件在安装 kubectl 命令行工具这一步中将会自动生成）拷贝到 node 节点的 `/etc/kubernetes/kubelet.kubeconfig` 位置，这样就不需要通过 CSR，当 kubelet 启动后就会自动加入的集群中。




