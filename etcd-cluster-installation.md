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
- 或者也可以用 yum 直接安装
- `yum -y install etcd`
- 若使用yum安装，默认 etcd 命令将在 `/usr/bin` 目录下，注意修改下面的 `etcd.service` 文件中的启动命令地址为 `/usr/bin/etcd`。

### 创建 etcd 的 systemd unit 文件
- 我们需要手动创建 `etcd` 的系统服务文件 `etcd.service`，修改后的文件如下所示：
  > 注意替换 IP 地址为你自己的 etcd 集群的主机 IP。
- `cat /usr/lib/systemd/system/etcd.service`

  ``` bash
  [Unit]
  Description=Etcd Server
  After=network.target
  After=network-online.target
  Wants=network-online.target
  Documentation=https://github.com/coreos
  
  [Service]
  Type=notify
  WorkingDirectory=/var/lib/etcd/
  EnvironmentFile=-/etc/etcd/etcd.conf
  ExecStart=/usr/local/bin/etcd \
    --name ${ETCD_NAME} \
    --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
    --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
    --peer-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
    --peer-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
    --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
    --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
    --initial-advertise-peer-urls ${ETCD_INITIAL_ADVERTISE_PEER_URLS} \
    --listen-peer-urls ${ETCD_LISTEN_PEER_URLS} \
    --listen-client-urls ${ETCD_LISTEN_CLIENT_URLS},http://127.0.0.1:2379 \
    --advertise-client-urls ${ETCD_ADVERTISE_CLIENT_URLS} \
    --initial-cluster-token ${ETCD_INITIAL_CLUSTER_TOKEN} \
    --initial-cluster infra1=https://192.168.8.66:2380,infra2=https://192.168.8.67:2380,infra3=https://192.168.8.68:2380 \
    --initial-cluster-state new \
    --data-dir=${ETCD_DATA_DIR}
  Restart=on-failure
  RestartSec=5
  LimitNOFILE=65536
  
  [Install]
  WantedBy=multi-user.target
  ```
- 完整 `Systemd Unit` 文件参见 [etcd.service](../kubernetes-manifests/systemd/etcd/etcd.service)



