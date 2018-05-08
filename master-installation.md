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

### 部署msater节点相关服务
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
