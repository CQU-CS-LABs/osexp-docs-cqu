## 1. WSL2的Ubuntu Docker无法启动

这个问题我们可以通过以下命令确认：

```bash
sudo service docker status
```

如果看到not running说明没有启动。

这个时候我们可以通过`dockerd -D`命令查看没有启动的原因，上网搜索相关资料。

同学们通常使用Ubuntu 22.04遇到的问题是Docker不支持新版iptables，可以通过以下命令解决：

```bash
sudo apt install iptables
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
sudo service docker start
```

## 2. WSL2无法联网

建议大家避免使用特殊的网络环境（如VPN），并重启电脑后再试。

## 3. apt update时出现the public key is not available

大家想想我们加Docker源的时候什么时候涉及过密钥？确保网络正常的情况下重新执行那条命令试试看。

## 4. apt install后出现failed to retireve available kernel versions.

与之俱来的还有failed to check for processor microcode upgrades.

这个是因为WSL2的Kernel被WSL自身管理，虚拟机也不需要microcode更新，因此这个提示可以忽略。

## 5. make时出现command not found

同学你进入Docker了吗？

## 6. 虚拟机图形操作不流畅

虚拟机显卡性能较低，大家可以在虚拟机外面装VSCode，然后remote SSH连到虚拟机。