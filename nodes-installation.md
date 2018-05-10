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

