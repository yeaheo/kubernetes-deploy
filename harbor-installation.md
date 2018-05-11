## kubernetes实践-部署 harbor 私有镜像仓库
- 配置私有 docker 镜像仓库 harbor，为的是将自己构建的镜像 push 到私有镜像仓库中，方便以后拉取。
- Habor软件Github官方网站：<https://github.com/vmware/harbor>
- Harbor安装部署详情可以参考：[habor安装部署](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)

### 安装 docker 和 docker-compose
- 安装 harbor 需要 docker 版本大于 `v1.10`，docker-compose 版本需要是 `v1.6` 以上。
- 安装 docker 可以参考：[docker-installation](https://docs.docker.com/engine/installation)
- 安装 docker 也可以参考个人笔记：[docker-ce-installation](https://github.com/yeaheo/hello.linux/blob/master/Docker/Docker-%E5%AE%89%E8%A3%85%E9%83%A8%E7%BD%B2.md)

- docker-compose 安装请参考：[docker-compose-installation](https://docs.docker.com/compose/install)
- 另一种安装 docker-compose 的方式是通过 `pip` 来安装，具体如下：
  ``` bash
  yum -y install epel-release
  yum -y install python-pip
  pip install docker-compose
  ```

### 安装 harbor 服务
- harbor 安装方式有两种，一种是在线安装，另一种是离线安装，为了可以快速安装我们选择离线安装，因为在线安装会在线下载所需镜像，速度较慢。
- **下载离线安装包**
- harbor下载地址：<https://github.com/vmware/harbor/releases>，在这里可以选择自己所需要的安装包进行下载。

- harbor 服务具体安装步骤如下：
  ``` bash
  1、下载并解压软件包;
  2、配置 `harbor.cfg` 文件;
  3、执行脚本文件 `install.sh` 进行安装;
  ```
  
- **解压软件包**
- `tar xvf harbor-online-installer-<version>.tgz -C /srv/`

- **修改 `harbor.cfg` 文件**
- 在这里，我们只需要修改以下参数：
    ``` cfg
    ......
    hostname = 192.168.8.69
    ......
    harbor_admin_password = admin123.
    ......
    ```
  > 注意： harbor.cfg 只需要修改 hostname 为你自己的机器 IP 或者域名，harbor 默认的 db 连接密码为 root123，可以自己修改，也可以保持默认，harbor 初始管理员密码为 Harbor12345，可以根据自己需要进行修改，email 选项是用来忘记密码重置用的，根据实际情况修改，如果使用 163 或者 qq 邮箱等，需要使用授权码进行登录，此时就不能使用密码登录了，否则会提示无效。

- **执行安装脚本**
  ``` bash
  cd /srv/harbor
  ./install.sh
  ```
- 等待安装过程即可。

- 待脚本执行完成后使用 docke-compose ps 即可查看，常用命令包含以下几个：  
- `docker-compose up -d`   后台启动，如果容器不存在根据镜像自动创建
- `docker-compose down -v` 停止容器并删除容器
- `docker-compose start`   启动容器，容器不存在就无法启动，不会自动创建镜像
- `docker-compose stop`    停止容器
  
  > 注：其实上面是停止 `docker-compose.yml` 中定义的所有容器，默认情况下 `docker-compose`就是操作同目录下的 `docker-compose.yml`文件，所以我们需要切换到 harbor 目录下才可以执行 docker-compose 相关命令，如果使用其他 `yml`文件，可以使用-f自己指定。

- 至此，harbor基本安装完成！

### 访问 harbor 服务
- 安装完成，可以通过浏览器输入地址 <http://192.168.8.69> 即可访问。
- 登陆账号/密码为：`admin/$harbor_admin_password`

### 配置 docker 访问 harbor
- docker 访问私有镜像仓库默认是通过 https 方式访问的，当我们用`docker login 192.168.8.69`登陆时会报错，如下：
  ``` bash
  Error response from daemon: Get https://192.168.8.69/v2/: dial tcp 192.168.8.69
  ```

- 当遇到上述问题后我们需要配置 docker：
- 修改 docker 配置文件，添加 docker 启动相关文件 `/etc/docker/daemon.json`
- `vim /etc/docker/daemon.json`
- 添加如下内容:
  ``` json
  { "insecure-registries":["192.168.8.69"] }
  ```
- 重启 docker 服务即可。
- `systemctl restart docker.service`
- 在做测试的时候，重启docker后发现依然报错，提示“拒绝访问”,后来发现80端口不是开启状态，需要重新启动 harbor
  ``` bash
  docker-compose stop
  docker-compose start
  ```
- 至此，所有操作基本完成！

### 验证 harbor
- 登陆：
  ``` bash
  [root@test-node2 ~]# docker login 192.168.8.69
  Username (admin): admin
  Password: 
  Login Succeeded
  ```
- push镜像
  ``` bash
  docker tag hello-world:latest 192.168.8.69/library/hello-world:v1
  docker push 192.168.8.69/library/hello-world:v1
  ```
- pull镜像
  ``` bash
  docker pull 192.168.8.69/library/hello-world:v1
  ```
  > docker 默认从 docker hub 拉取镜像，在国内速度比较慢，需要配置镜像速度器，具体参见 [docker配置镜像加速器](https://github.com/yeaheo/hello.linux/blob/master/Docker/Docker-%E5%8A%A0%E9%80%9F%E5%99%A8.md)

