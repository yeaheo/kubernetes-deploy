## kubernetes实践-创建etcd集群
- kuberntes 集群使用 etcd 存储所有数据,本部分我们介绍部署一个三节点的高可用 etcd 集群，这三个节点复用kubernetes master机器。
- 三个 etcd 节点如下所示：
  | role   | IP 地址 |
  -------- | -------------- 
  | etcd-0 | 192.168.8.66 |
  | etcd-1 | 192.168.8.67 |
  | etcd-2 | 192.168.8.68 |

