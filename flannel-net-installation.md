## kubernetes实践-flannel及docker服务配置

- **检查 TLS 证书**
  ``` bash
  [root@ceph-node1 ~]# ls /etc/kubernetes/ssl/
  admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem
  ```

### 配置 Flannel 服务
- 建议直接使用 yum 安装 flanneld，除非对版本有特殊需求，默认安装的是 `v0.7.1` 版本的 flannel。
- 安装 flanneld 服务，具体参考如下：
  ``` bash
  yum -y install flannel
  systemctl start flanneld.service
  ```
  > 在起 flanneld 服务之前，我们需要修改 flanneld服务的 Systemd Unit 文件和对应的配置文件

- **修改 flannel 服务的 systemd unit 文件**
- `cat /usr/lib/systemd/system/flanneld.service`
  ``` bash
  [Unit]
  Description=Flanneld overlay address etcd agent
  After=network.target
  After=network-online.target
  Wants=network-online.target
  After=etcd.service
  Before=docker.service
  
  [Service]
  Type=notify
  EnvironmentFile=/etc/sysconfig/flanneld
  EnvironmentFile=-/etc/sysconfig/docker-network
  ExecStart=/usr/bin/flanneld-start \
      $FLANNEL_OPTIONS \
      -etcd-endpoints=${FLANNEL_ETCD_ENDPOINTS} \
      -etcd-prefix=${FLANNEL_ETCD_PREFIX}
  ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
  Restart=on-failure
  
  [Install]
  WantedBy=multi-user.target
  RequiredBy=docker.service
  ```
- 完整 flanneld systemd unit 参见 [flanneld.service](https://github.com/yeaheo/kubernetes-manifests/blob/master/systemd/flanneld.service)

- **修改 flannel 服务配置文件**
- flannel 服务配置文件: `/etc/sysconfig/flanneld`
  ``` bash
  # Flanneld configuration options  
  
  # etcd url location.  Point this to the server where etcd runs
  #FLANNEL_ETCD_ENDPOINTS="http://127.0.0.1:2379"
  FLANNEL_ETCD_ENDPOINTS="https://192.168.8.66:2379,https://192.168.8.67:2379,https://192.168.8.68:2379"
  
  # etcd config key.  This is the configuration key that flannel queries
  # For address range assignment
  #FLANNEL_ETCD_PREFIX="/atomic.io/network"
  FLANNEL_ETCD_PREFIX="/kube-centos/network"
  
  # Any additional options that you want to pass
  #FLANNEL_OPTIONS=""
  FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"
  ```
- 完整 flannel 服务配置文件参见： [flanneld](https://github.com/yeaheo/kubernetes-manifests/blob/master/config/flanneld)

  > flannel 服务的 systemd unit 文件及配置文件修改完成后，就可以启动 flanneld 服务了。
  ``` bash
  systemctl daemon-reload
  systemctl start flanneld.service
  ```

- **在 etcd 中创建网络配置**
- 执行下面的命令为docker分配IP地址段:
  ``` bash
  etcdctl --endpoints=https://192.168.8.66:2379,https://192.168.8.67:2379,https://192.168.8.68:2379 \
    --ca-file=/etc/kubernetes/ssl/ca.pem \
    --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
    --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
    mkdir /kube-centos/network

  etcdctl --endpoints=https://192.168.8.66:2379,https://192.168.8.67:2379,https://192.168.8.68:2379 \
    --ca-file=/etc/kubernetes/ssl/ca.pem \
    --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
    --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
    mk /kube-centos/network/config '{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'
  ```

### 配置 docker 服务
- docker 需要和 flannel 服务在同一个网段内，这样后期才能互通。
- docker 服务的安装直接用 yum 包管理工具直接安装即可：
  ``` bash
  yum -y install docker
  ```
- 配置 docker 服务如下：
  ``` bash
  systemctl stop docker.service
  source  /run/flannel/subnet.env 
  docker  daemon  --bip=${FLANNEL_SUBNET}  --mtu=${FLANNEL_MTU} &
  ifconfig docker0 $FLANNEL_SUBNET
  ```
- 修改docker服务文件增加如下内容：
  ``` bash
  EnvironmentFile=-/run/flannel/docker 
  EnvironmentFile=-/run/docker_opts.env 
  EnvironmentFile=-/run/flannel/subnet.env
  ```
- 配置完成后需要重启 docker 及 flannel 服务。
  ``` bash
  systemctl daemon-reload
  systemctl restart docker.service
  systemctl restart flanneld.service
  ```
  > 我们在重启 docker 服务的时候可能重启失败，原因一般是 docker 进程还存在，将其杀死再重启既可，可以参考：`ps axf | grep docker | grep -v grep | awk '{print "kill -9 " $1}' | sudo sh`

### 查询 etcd 中的内容
- docker 和 flannel 服务启动正常后，我们可以参照下面的方式查看 etcd 的内容：
  ``` bash
  [root@k8s-master ~]# etcdctl --endpoints=${ETCD_ENDPOINTS} \
      --ca-file=/etc/kubernetes/ssl/ca.pem \
      --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
      --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
      ls /kube-centos/network/subnets
  /kube-centos/network/subnets/172.30.78.0-24
  /kube-centos/network/subnets/172.30.79.0-24
  /kube-centos/network/subnets/172.30.41.0-24
  ```

  > 其实 flanneld 和 docker 服务在 master 节点上是非必需的，但是我们建议在 master 节点上同样配置 flanneld 和 docker 服务，因为这样可以很方便的通过 master 节点访问 node 上的服务，例如插件 dashboard 的访问，我们可以通过 api-server 来访问：<http://master-ip:8080/ui>。
  > master 节点的 flanneld 和 docker 服务安装好后，我们在 master 节点可以 ping 通 nodes 节点的 flanneld 地址，例如：
  ``` bash
    [root@k8s-master ~]# ping 172.30.41.0
    PING 172.30.41.0 (172.30.41.0) 56(84) bytes of data.
    64 bytes from 172.30.41.0: icmp_seq=1 ttl=64 time=0.081 ms
    ^C
    --- 172.30.41.0 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 0.081/0.081/0.081/0.000 ms
    [root@k8s-master ~]# ping 172.30.78.0
    PING 172.30.78.0 (172.30.78.0) 56(84) bytes of data.
    64 bytes from 172.30.78.0: icmp_seq=1 ttl=64 time=0.648 ms
    ^C
    --- 172.30.78.0 ping statistics ---
    1 packets transmitted, 1 received, 0% packet loss, time 0ms
    rtt min/avg/max/mdev = 0.648/0.648/0.648/0.000 ms
    [root@k8s-master ~]# ping 172.30.79.0
    PING 172.30.79.0 (172.30.79.0) 56(84) bytes of data.
    64 bytes from 172.30.79.0: icmp_seq=1 ttl=64 time=0.658 ms
  ```







