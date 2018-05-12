## kubernetes实践-安装 heapster 插件
- 当我们安装了 `kubernetes-dashboard` 后默认是不显示 CPU 负载信息折线图等信息的。需要我们安装 heapster 插件。
- heapster 软件下载地址：<https://github.com/kubernetes/heapster/releases>，我们可以在这里下载适合我们的版本。

### 准备 heapster 的 yaml 文件
- 本次实验我们下载 `v1.5.2` 版本。
  ``` bash
  wget https://codeload.github.com/kubernetes/heapster/zip/v1.5.2
  unzip -d /usr/src/ heapster-1.5.2.zip
  ```
- 默认的 yaml 路径为： `/usr/src/heapster-1.5.2/deploy/kube-config/influxdb`
  > 我将压缩包解压到了 `/usr/src` 目录下了。
  
  ``` bash
  # pwd
  /usr/src/heapster-1.5.2/deploy/kube-config/influxdb
  
  # ls
  grafana.yaml  heapster.yaml  influxdb.yaml
  ```
- 默认情况下，上述三个文件内定义的镜像都是官方镜像：
  ``` bash
  [root@k8s-master influxdb]# grep -rn image | grep gcr
  grafana.yaml:16:        image: gcr.io/google_containers/heapster-grafana-amd64:v4.4.3
  heapster.yaml:23:        image: gcr.io/google_containers/heapster-amd64:v1.5.1
  influxdb.yaml:16:        image: gcr.io/google_containers/heapster-influxdb-amd64:v1.3.3
  ```
- 官方镜像保存在 `gcr.io` 中需要翻墙才能下载，为了方便其他人使用，我下载了一份并上传到了时速云镜像市场，供大家下载使用。
  ``` bash
  index.tenxcloud.com/yeaheo/heapster-grafana-amd64:v4.4.3
  index.tenxcloud.com/yeaheo/heapster-amd64:v1.5.1
  index.tenxcloud.com/yeaheo/heapster-influxdb-amd64:v1.3.3
  ```
- 还是建议将这三个镜像下载下来传到我们的 harbor 私有镜像仓库中。

- 修改好的 yaml 文件参见: [heapster](https://github.com/yeaheo/kubernetes-manifests/tree/master/addons/heaspter)

### 执行相关 yaml 文件
- 修改完成后，需要执行这三个文件：
  ``` bash
  cd /opt/k8s-addons/heapster
  kubectl create -f .
  ```
- **检查执行结果**:
  ``` bash
  # kubectl get deployments -n kube-system | grep -E 'heapster|monitoring'
  heapster               1         1         1            1           16d
  monitoring-grafana     1         1         1            1           16d
  monitoring-influxdb    1         1         1            1           16d

  # kubectl get pods -n kube-system | grep -E 'heapster|monitoring'
  heapster-dd67994cd-6jr58                1/1       Running   0          16d
  monitoring-grafana-5d87678f85-fwlpf     1/1       Running   0          16d
  monitoring-influxdb-c96b6ffbb-75lns     1/1       Running   0          16d
  ```
- 检查 kubernets dashboard 界面，看是显示各 Nodes、Pods 的 CPU、内存、负载等利用率曲线图，如果没有出现可以重启一下 dashboard。

- 我们可以用以下命令查看集群信息：
  ``` bash
  # kubectl cluster-info 
  Kubernetes master is running at https://192.168.8.66:6443
  Heapster is running at https://192.168.8.66:6443/api/v1/namespaces/kube-system/services/heapster/proxy
  KubeDNS is running at https://192.168.8.66:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
  monitoring-grafana is running at https://192.168.8.66:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
  monitoring-influxdb is running at https://192.168.8.66:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy
  ```
- 这些都是可以访问到的，这里不再赘述，还请自行访问测试。
