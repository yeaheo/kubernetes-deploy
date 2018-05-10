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



