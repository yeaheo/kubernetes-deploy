## kubernetes实践-创建etcd集群
- kuberntes 集群使用 etcd 存储所有数据,本部分我们介绍部署一个三节点的高可用 etcd 集群，这三个节点复用kubernetes master机器。
- 三个 etcd 节点如下所示：
  ``` bash
  etcd-0 | 192.168.8.66 
  etcd-1 | 192.168.8.67 
  etcd-2 | 192.168.8.68 
  ```
- **TLS认证文件**
- 在这里，我们需要为 etcd 集群创建加密通信的 TLS 证书，为了方便我们在这里复用以前创建的 kubernetes 证书。具体证书配置如下所示：
  ``` bash
  cd /root/ssl
  cp ca.pem kubernetes-key.pem kubernetes.pem /etc/kubernetes/ssl
  ```
  > 注意：
  > kubernetes 证书的 hosts 字段列表中包含上面三台机器的 IP，否则后续证书校验会失败。

### 部署 etcd 集群
- `etcd` 软件下载地址：<https://github.com/coreos/etcd/releases>，我们可以下载最新版的 `etcd` 软件包。
- 下载并安装 etcd 二进制软件包
  ``` bash
  wget https://github.com/coreos/etcd/releases/download/v3.2.18/etcd-v3.2.18-linux-amd64.tar.gz
  tar -xvf etcd-v3.2.18-linux-amd64.tar.gz
  mv etcd-v3.2.18-linux-amd64/etcd* /usr/local/bin
  ```
### 创建 etcd 的 systemd unit 文件
  > 注意替换 IP 地址为你自己的etcd集群的主机IP。
- 
- `cat /usr/lib/systemd/system/etcd.service`

