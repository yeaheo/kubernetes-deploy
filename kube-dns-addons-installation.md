## kubernetes实践-安装 kube-dns 插件
- `kube-dns` 官方的 `yaml` 文件其实在我们先前下载的 `kubernetes server` 软件包内，具体路径为：`/srv/kubernetes/cluster/addons/dns`
  
  > 我是把 kuberntes 解压到了 `/srv/` 目录下

### 准备 kube-dns 相关镜像

- `kube-dns` 插件直接使用 kubernetes 部署，官方的配置文件中包含以下镜像：
  
  ``` bash
  [root@k8s-master dns]# cat kube-dns.yaml.base | grep image
        image: gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.8
        image: gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.8
        image: gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.8
  [root@k8s-master dns]# pwd
  /srv/kubernetes/cluster/addons/dns
  ```
- 因为墙的原因，这些镜像我们需要翻墙下载，很是不方便，我下载了所需要的镜像，并上传到了时速云镜像仓库欢迎大家下载使用：
  ``` bash
  index.tenxcloud.com/yeaheo/k8s-dns-dnsmasq-nanny-amd64:1.14.8
  index.tenxcloud.com/yeaheo/k8s-dns-kube-dns-amd64:1.14.8
  index.tenxcloud.com/yeaheo/k8s-dns-sidecar-amd64:1.14.8
  ```
- 我们 `pull` 下来相关镜像后建议上传到私有仓库中，这样下载速度会更快，更方便，私有仓库部署参见 [harbor私有镜像仓库部署](./harbor-installation.md)

### 准备 kube-dns 相关 yaml 文件
- 默认情况下 kube-dns 插件的 `yaml` 文件在对应目录下已经存在，我们只需复制一份到指定目录下即可：
  ``` bash
  [root@k8s-master dns]# pwd
  /srv/kubernetes/cluster/addons/dns   
  [root@k8s-master dns]# cp kube-dns.yaml.base /opt/k8s-addons/dns/kube-dns.yaml
  ```
- 然后需要我们修改相关 `yaml` 文件。
- 这里我们用源文件 `kube-dns.yaml.base` 做修改，修改后的 `kube-dns.yaml` 与源文件的区别如下所示，而这些正式我们需要修改的：
  ``` bash
  [root@k8s-master dns]# diff kube-dns.yaml kube-dns.yaml.base 
  33c33
  <   clusterIP: 10.254.0.2
  ---
  >   clusterIP: __PILLAR__DNS__SERVER__
  97c97
  <         image: 192.168.8.69/library/k8s-dns-kube-dns-amd64:1.14.8
  ---
  >         image: gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.8
  127c127
  <         - --domain=cluster.local.
  ---
  >         - --domain=__PILLAR__DNS__DOMAIN__.
  148c148
  <         image: 192.168.8.69/library/k8s-dns-dnsmasq-nanny-amd64:1.14.8
  ---
  >         image: gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.8
  168c168
  <         - --server=/cluster.local./127.0.0.1#10053
  ---
  >         - --server=/__PILLAR__DNS__DOMAIN__/127.0.0.1#10053
  187c187
  <         image: 192.168.8.69/library/k8s-dns-sidecar-amd64:1.14.8
  ---
  >         image: gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.8
  200,201c200,201
  <         - --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local.,5,SRV
  <         - --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local.,5,SRV
  ---
  >         - --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,5,SRV
  >         - --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,5,SRV
  ```
  > yaml 配置文件中使用的是私有镜像仓库中的镜像。这些文件都是统一成为一个文件，当然也可以做分离，这样更直观些。

- 修改好的 kube-dns 相关 yaml 文件参见 [kube-dns.yaml](https://github.com/yeaheo/kubernetes-manifests/blob/master/addons/kube-dns/kube-dns.yaml)

### 系统预定义的 RoleBinding
- 预定义的 `RoleBinding system:kube-dns` 将 `kube-system` 命名空间的 `kube-dns ServiceAccount` 与 `ystem:kube-dns Role` 绑定， 该 Role 具有访问 `kube-apiserver` DNS 相关 API 的权限；
  
  ``` bash
  [root@k8s-master ~]# kubectl get clusterrolebindings system:kube-dns -o yaml
  apiVersion: rbac.authorization.k8s.io/v1
  kind: ClusterRoleBinding
  metadata:
    annotations:
      rbac.authorization.kubernetes.io/autoupdate: "true"
    creationTimestamp: 2018-04-06T09:06:30Z
    labels:
      kubernetes.io/bootstrapping: rbac-defaults
    name: system:kube-dns
    resourceVersion: "87"
    selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Akube-dns
    uid: cbda2173-3979-11e8-8d8b-525400472b24
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: system:kube-dns
  subjects:
  - kind: ServiceAccount
    name: kube-dns
    namespace: kube-system
  ```
- `kube-dns.yaml` 里定义的 pods 时使用了 `kube-dns` 的 `ServiceAccount` 所以具有访问 `kube-apiserver` DNS 相关 API 的权限。
- 其他不用配置，例如 `kube-dns` 的 `ServiceAccount` 我们默认就好。

### 配置 kube-dns 相关服务
- 之前我们在修改 `kube-dns.yaml` 文件的时候，有如下一处修改：
  ``` bash
  33c33
  <   clusterIP: 10.254.0.2
  ---
  >   clusterIP: __PILLAR__DNS__SERVER__
  ```
- `spec.clusterIP = 10.254.0.2`，明确指定了 `kube-dns Service IP`，这个 IP 需要和 kubelet 的 `--cluster-dns` 参数值一致，否则启动 `kube-dns` 的时候会报错；

- **执行 `kube-dns` 定义文件:**
  ``` bash
  [root@k8s-master ~]# cd /opt/k8s-addons/dns/
  [root@k8s-master dns]# ll
  total 8
  -rw-r--r-- 1 root root 6051 Apr 26 10:47 kube-dns.yaml
  [root@k8s-master dns]# kubectl create -f kube-dns.yaml
  ```
- **验证 `kube-dns` 相关服务**
  ``` bash
  [root@k8s-master dns]# kubectl get pods -n kube-system | grep dns
  kube-dns-559bc869fb-tzc2b               3/3       Running   0          15d
  
  [root@k8s-master dns]# kubectl get svc -n kube-system | grep dns
  kube-dns               ClusterIP   10.254.0.2       <none>        53/UDP,53/TCP    15d
  ```
- 可以看出， `kube-dns` 插件已经安装完成，相关 `sevice` 及 `pod` 都在正常工作。

### 检查 kube-dns 功能
- `kube-dns` 插件安装完成后我们需要验证一下 `kube-dns` 的相关功能。
- 新建一个 deployment 验证 kube-dns 功能：
  ``` bash
  [root@k8s-master nginx]# cat my-nginx.yaml 
  
  apiVersion: extensions/v1beta1
  kind: Deployment
  metadata:
    name: my-nginx
  spec:
    replicas: 2
    template:
      metadata:
        labels:
          run: my-nginx
      spec:
        containers:
        - name: my-nginx
          image: 192.168.8.69/library/nginx:1.11.2
          ports:
          - containerPort: 80
  ```
- export 该 deployment, 生成 my-nginx 服务:
  ``` bash
  # kubectl create -f my-nginx.yaml 
  deployment "my-nginx" created
  # kubectl expose deploy my-nginx
  service "my-nginx" exposed
  # kubectl get services --all-namespaces |grep my-nginx
  default       my-nginx               ClusterIP   10.254.196.99    <none>        80/TCP           6s
  ```
- 创建另一个 pod，查看 `/etc/resolv.conf` 是否包含 `kubelet` 配置的 `--cluster-dns` 和 `--cluster-domain`，是否能够将服务 `my-nginx` 解析到 Cluster IP `10.254.196.99`。

  ``` bash
  $ kubectl create -f nginx-pod.yaml
  $ kubectl exec  nginx -i -t -- /bin/bash
  root@nginx:/# cat /etc/resolv.conf
  nameserver 10.254.0.2
  search default.svc.cluster.local. svc.cluster.local. cluster.local. jimmysong.io
  options ndots:5
  
  root@nginx:/# ping my-nginx
  PING my-nginx.default.svc.cluster.local (10.254.196.99): 56 data bytes
  76 bytes from 119.147.223.109: Destination Net Unreachable
  ^C--- my-nginx.default.svc.cluster.local ping statistics ---
  
  root@nginx:/# ping kubernetes
  PING kubernetes.default.svc.cluster.local (10.254.0.1): 56 data bytes
  ^C--- kubernetes.default.svc.cluster.local ping statistics ---
  11 packets transmitted, 0 packets received, 100% packet loss
  
  root@nginx:/# ping kube-dns.kube-system.svc.cluster.local
  PING kube-dns.kube-system.svc.cluster.local (10.254.0.2): 56 data bytes
  ^C--- kube-dns.kube-system.svc.cluster.local ping statistics ---
  6 packets transmitted, 0 packets received, 100% packet loss
  ```
- 从结果来看，service名称可以正常解析， 说明 kube-dns 可以正常工作。
  > 直接 `ping ClusterIP` 是 `ping` 不通的，`ClusterIP` 是根据 `iptables` 路由到服务的 `endpoint`上，只有结合 `ClusterIP`加端口才能访问到对应的服务。

