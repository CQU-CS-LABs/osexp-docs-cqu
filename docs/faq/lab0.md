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

## 7. WSL无法安装，提示要开启虚拟化，但是虚拟化已经开启

很有可能是有些同学用了一些无法和Windows虚拟化平台共存的软件导致的。例如部分Android模拟器（为什么不使用 Windows 11 自带的性能更好的 Windows Subsystem for Android替代呢？）。

这些软件会修改bcd禁用虚拟化，我们可以使用以下命令重新开启：

```powershell
bcdedit /set hypervisorlaunchtype auto
```

## 8. WSL2里安装了Docker，VSCode找不到

VSCode不能连WSL2里的Docker，需要先使用WSL Remote连到WSL里的VSCode，再在这个里面装VSCode的Docker插件。

## 9. apt的https出现证书错误

一般有两种情况：

1. 虚拟机网卡用了桥接模式，但是虚拟机没有登录校园网。

    解决方法：改为NAT模式，共享你电脑的IP地址上网。

2. 使用的Ubuntu版本较老，CA证书没有更新

    修改`/etc/apt/sources.list`，将所有的https改为http，然后重新执行`apt update`与`apt upgrade`。

## 10. 按照清华镜像站安装Docker无法添加gpg密钥，提示no such file or directory

还是Ubuntu版本问题，Ubuntu 20.04没有`/etc/apt/keyrings`文件夹，需要大家自行使用`mkdir -p /etc/apt/keyrings`创建。

## 来自助教的提醒

最后，还是希望大家能自己读懂错误提示，培养自己解决问题的能力。

别人根据你的错误告诉你解决方案只不过是治标不治本，当同学们以后做的工作基于一些比较新的框架以及开源代码时可能还时常需要修这些开源代码本身的问题，甚至经常会给这些开源项目提交Patch。诚然，解决问题的过程需要很多的基础知识以及经验，但希望大家能够在遇到这种问题时，能够自己找到解决问题的思路。

给同学一些建议：

1. 学会读英文的错误提示，遇到错误是正常的，不要害怕。

2. 学会根据提示上网搜索，并尝试用英语以及搜索英语较为友好的搜索引擎。

    某些搜索引擎对于某些网站SEO太过靠前很容易搜到一些不专业的内容。让你无法了解问题的本质，导致一些过时的问题。
    
    诸如Ubuntu换了一个旧版本的源然后用aptitude或者apt install -f把整个系统的软件包依赖完全破坏，最后只能重装系统。然而用aptitude和apt install -f本身就是非常错误的做法。

3. 学会读软件的文档。以及Linux下各种工具的man。当无法读文档解决问题时，或许我们还要自己去阅读相关的代码。