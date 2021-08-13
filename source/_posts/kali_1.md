```
kali学习日记
```

开机操作O(∩_∩)O



```
sudo passwd root 设置密码
su root 
pwd
su -root
sudo -i
```

```
echo $SHELL(/usr/bin/zsh) ---记录当前shell(bash/zsh)
```

配镜像源

```
vim /etc/apt/sources.list
```

```
(vim):set nu
```

中科大镜像源

```shell
deb http://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib
```



```
deb-src http://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib
```

复制粘贴kali:选中+滚轮

deb 代表软件的位置 deb-src 代表源代码所在的位置

```
windows .exe .zip .rar

linux .rpm  .tar.gz .deb
```

更新前apt update

kali 的 apt 源

Kali Rolling :是kali即时更新版，只要kali中有更新，更新包就会放入Kali Rolling中

在kali rolling 下有三类软件包 main 、non-free、contrib;

| dists区域 | 软件包组件标准                                          |
| --------- | ------------------------------------------------------- |
| main      | 遵从Debian 自由软件指导方针(DFSG)，并且不依赖于non-free |
| contrib   | 遵从Debian 自由软件指导方针(DFSG)，但依赖于non-free     |
| non-free  | 不遵从Debian 自由软件指导方针(DFSG)                     |

apt  apt-get  包含关系

apt常用命令：

```
apt install  安装软件包
apt remove  移除软件包
apt update  更新可用软件包列表
apt upgrade  通过 安装/升级 软件来更新系统
apt dist-upgrade (有风险)带卸载的更新
vim /etc/apt/sources.list 编辑软件源信息文件
```

init 0 关机

ifconfig 

ifconfig eth0 192.168.1.53/24(子网掩码255.255.255.0)   配置临时ip

route add default gw 192.168.1.1       配置默认路由

ping 192.168.1.1

vim /etc/resolv.conf

echo nameserver 8.8.8.8 >/etc/resolv.conf    配置dns服务

永久搞一个ip

vim /etc/network/interfaces

添加：

```
auto eth0

#iface eth0 inet dhcp

iface eth0 inet static     静态地址

address 192.168.1.53	   固定IP地址

netmask 255.255.255.0      子网掩码

gateway 192.168.1.1        网关
```

机器

```
kali xp win10/7 centos7 靶机
```

重启网络服务

 /etc/init.d/networking                             /etc/init.d/networking restart

systemctl  restart networking                 systemctl  restart networking.service        

vim /etc/ssh/sshd_config

找#Authentication:

第二行  PermitRootLogin yes

三行后 PubkeyAuthentication yes

 /etc/init.d/ssh restart     /  systemctl  restart  ssh

update -rc.d ssh enable



xshell 连接 

传文件 yum install lrzsz-y

apt install lrzsz

rz 上传

sz 下载

搬瓦工

国外vps服务器

350https://bwh81.net/cart.php

*** 利用第三方服务对目标进行被动信息收集，防止被发现