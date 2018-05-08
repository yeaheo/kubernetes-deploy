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


