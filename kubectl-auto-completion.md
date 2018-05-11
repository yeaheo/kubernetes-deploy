## kubectl命令自动补全

- 成功部署了 kubernetes 集群后，我们通常是通过 kubectl 这个命令行工具进行操作，默认该工具不能自动补全命令，但是我们可以进行一系列配置来实现其自动补全的功能，kubectl 命令行工具本身就支持 complication ，只需要简单的设置下就可以了。

- 以下是linux系统的设置命令：
  ``` bash
  source <(kubectl completion bash)
  echo "source <(kubectl completion bash)" >> ~/.bashrc
  ```
- 然后就可以自动补全了。
- 如果是普通用户上述命令依旧可用，复制执行即可！
- 如果发现不能自动补全，可以尝试安装 `bash-completion` 软件包，然后刷新即可！
  ``` bash
  yum -y install bash-completion
  ```
