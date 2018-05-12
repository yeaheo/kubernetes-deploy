## kubernetes实践-安装 dashboard 插件
- Kubernetes 仪表板是 Kubernetes 集群的通用基于 Web 的 UI。它允许用户管理在集群中运行的应用程序，对它们进行故障排除，并管理集群本身。
- Kube-dashboard 的官方文件参见 [Kubernetes-dashboard](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dashboard) 
- 想更深入的了解 Kubernetes-dashboard 可以参考：<https://github.com/kubernetes/dashboard>

- 自 `v1.7` 版 Dashboard 使用更安全的设置。这意味着，默认情况下，它拥有最少的一组特权，只能通过 HTTPS 访问。在执行任何进一步的步骤之前，建议阅读访问控制指南。

- 我们在安装 Kubernetes-dashboard 的时候，可以用官方的 `yaml` 文件, 也可以通过下面展示的方式：
  ``` bash
  $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
  ```
- 其实，这两种方式都是一样的，只不过官方的 `yaml` 文件都按照 `kind` 分离出来了。

### 准备 Kubernetes-dashboard 定义文件
- **下载最新版 yaml 文件：**
  ``` bash
  cd /opt/k8s-addons/dashboard/
  wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
  ```
- **修改 yaml 文件**
- 默认情况下，kubernetes-dashboard 的定义文件中说明的 dashboard 相关镜像都是谷歌上，具体如下：
  ``` bash
  [root@k8s-master dashboard]# cat kubernetes-dashboard.yaml | grep image
  image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
  ```
- 由于网络的问题，我下载了原版镜像并上传到时速云的镜像仓库，可参考如下进行下载：
  ``` bash
  # docker pull index.tenxcloud.com/yeaheo/kubernetes-dashboard-amd64:1.8.3
  ```
- 但是还是建议将该镜像上传至集群的私有镜像仓库，这样会很方便，而且拉取镜像速度比较快，私有仓库部署参见 [harbor私有镜像仓库部署](./harbor-installation.md)

- 将service type设置为 `NodePort`，修改后的 `yaml` 文件见 [kubernetes-dashboard.yaml](https://github.com/yeaheo/kubernetes-manifests/blob/master/addons/dashboard/kubernetes-dashboard.yaml)。
  > 镜像文件文档里写的是私有仓库的地址。

### 安装 dashboard 插件
- 修改完 yaml 文件后，然后需要执行该文件：
  ``` bash
  # kubectl create -f kubernetes-dashboard.yaml 
  secret "kubernetes-dashboard-certs" created
  serviceaccount "kubernetes-dashboard" created
  role "kubernetes-dashboard-minimal" created
  rolebinding "kubernetes-dashboard-minimal" created
  deployment "kubernetes-dashboard" created
  service "kubernetes-dashboard" created
  ```
- 查看 dashboard 相关 pod 及 service ：
  ``` bash
  # kubectl get pods -n kube-system | grep dashboard
  kubernetes-dashboard-7dc8587bbf-vjq5w   1/1       Running   0          5m

  # kubectl get svc -n kube-system | grep dashboard
  kubernetes-dashboard   NodePort    10.254.119.206   <none>        443:32579/TCP    5m
  ```

### 访问 Kubernetes-dashboard
- dashboard 的访问方式一般有三种，分别是：
- kubernetes-dashboard 服务暴露了 `NodePort`，可以使用 <http://NodeIP:nodePort> 地址访问 dashboard；
- 通过 API server 访问 dashboard（https 6443端口和http 8080端口方式）；
- 通过 kubectl proxy 访问 dashboard；

- 为了方便，本次我们通过用 API server 访问 dashboard：<http://master-ip:8080/ui>

![dashboard-login](https://github.com/yeaheo/hello.linux/blob/master/images/dashboard-login.png)

- 由上图可知，新版 dashboard 支持使用 `kubeconfig` 和 `token` 两种的认证方式
- 这些认证问题，后续再研究，我们本次给匿名用户授予 `cluster-admin` 权限。

- **准备匿名用户相关权限 yaml 文件**
- `cat kubernetes-dashboard-admin.yaml`
  ``` yaml
  apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: ClusterRoleBinding
  metadata:
    name: kubernetes-dashboard
    labels:
      k8s-app: kubernetes-dashboard
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: cluster-admin
  subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system
  ```
- 完整 yaml 文件参见 [kubernetes-dashboard-admin.yaml](https://github.com/yeaheo/kubernetes-manifests/blob/master/addons/dashboard/ann-user-cluster-admin.yaml)

- 执行该 yaml 文件：
  ``` bash
  # kubectl create -f kubernetes-dashboard-admin.yaml
  ```
- 执行成功后，直接 skip 登录验证，就可以看到完整的 dashboard 的界面了。

![dashboard-ui](https://github.com/yeaheo/hello.linux/blob/master/images/dashboard-ui.png)

- 后续有关 Kubernetes-dashboard 的功能持续更新中...


