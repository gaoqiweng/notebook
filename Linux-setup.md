# Linux系统配置

本文档为Linux服务器的配置方面的笔记，Linux相关笔记还有：

[Linux命令行操作技巧](Linux-cli.md)

[SSH远程登录](Linux-SSH.md)

[Linux备份](Linux-backup.md)

TOC:


----

## 如何翻墙

## 部署shadowsocks客户端，并部署Privoxy提供http proxy

代码参见[ssprivoxy.txt](code/ssprivoxy.txt)

----

## 配置有线静态IP
```bash
vim /etc/network/interfaces
# 写入以下内容，请自行替换xx部分
iface eth0 inet static
 address 10.xx.xx.13 #你需要设置的IP
 netmask 255.255.255.0 #子网掩码
 network 10.xx.xx.0
 broadcast 10.xx.xx.255
 gateway 10.xx.xx.254 #网关
 dns-nameservers 10.10.0.21 #dns服务器
# 按Esc, :wq退出保存
service networking restart
ifconfig eth0 10.xx.xx.13 netmask 255.255.255.0 up
route add default eth0 #路由配置也很重要，错误的路由将导致不能访问
route add default gw 10.xx.xx.254 dev eth0 #这里设置为你的网关
```

注意使用ifconfig进行ip的修改后，会丢失路由信息、额外的ip设置，需要重新配置route（执行上述两条route命令即可）

## 配置为dhcp自动获取ip，解决RTNETLINK answers: File exists问题

之前已经配置过静态ip，现在要改为自动获取

```
dhclient eth0
```

出现报错RTNETLINK answers: File exists，解决方案：

```
ip addr flush dev eth0
```

## 配置apt源以加速国内环境下apt速度

    curl http://mirrors.163.com/.help/sources.list.trusty>/etc/apt/sources.list

如果还未安装curl，先手动写入这两行：

```
deb http://mirrors.163.com/ubuntu/ trusty main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ trusty-security main restricted universe multiverse
```

> 注：vim复制一行的命令为yy，粘贴为p

或者通过sed替换：

```
sed -i 's/security.ubuntu.com/mirrors.zju.edu.cn/g' /etc/apt/sources.list
sed -i 's/archive.ubuntu.com/mirrors.zju.edu.cn/g' /etc/apt/sources.list
```

## 单网卡获得多个IP
ifconfig eth0:233 10.xx.xx.233 netmask 255.255.255.0 up

----

## 锐速安装

来自：https://github.com/91yun/serverspeeder

安装之前需要修改内核版本并重启：

    apt-get install linux-image-3.16.0-43-generic
    reboot

安装命令：# 此安装脚本会连接开发者的服务器以root权限执行远程指令，风险自负

    wget -N --no-check-certificate https://raw.githubusercontent.com/91yun/serverspeeder/master/serverspeeder-all.sh && bash serverspeeder-all.sh
    
查看状态/关闭服务：

    service serverSpeeder stauts
    service serverSpeeder stop
    
----

## 解决apt依赖问题

问题描述：服务器为ubuntu14.04版本，某些不明操作后，无法用`apt-get`安装任何东西

```bash
> apt-get -f install
Reading package lists... Done
Building dependency tree
Reading state information... Done
Correcting dependencies... failed.
The following packages have unmet dependencies:
 libatk1.0-0 : Depends: libglib2.0-0 (>= 2.41.1) but 2.40.0-2 is installed
 libglib2.0-bin : Depends: libglib2.0-0 (= 2.44.0-1ubuntu3) but 2.40.0-2 is installed
 libglib2.0-dev : Depends: libglib2.0-0 (= 2.44.0-1ubuntu3) but 2.40.0-2 is installed
 libgtk2.0-0 : Depends: libglib2.0-0 (>= 2.41.1) but 2.40.0-2 is installed
E: Error, pkgProblemResolver::Resolve generated breaks, this may be caused by held packages.
E: Unable to correct dependencies
```

仔细看错误说明，libglib2.0-bin这个软件包要求libglib2.0-0的版本=2.44但是现有的安装版本为2.40

在ubuntu的软件包官网搜索咯：https://launchpad.net/ubuntu/

发现2.44版本的是vivid才提供的，现在系统版本是trusty，自然apt-get装不了

解决方案：

找到报错信息需要的精确匹配的那个deb文件下载咯，例如这里就要下载这个版本的：

https://launchpad.net/ubuntu/vivid/amd64/libglib2.0-0/2.44.0-1ubuntu3

得到deb文件后`dpkg -i 文件名`

### Note

一般apt依赖冲突问题都是由于系统版本与需要的包的版本不一致导致的，检查一下/etc/apt/sources.list看看是否匹配系统版本咯

用apt-get前检查一下sources.list，树莓派是版本8，是jessie不是wheezy!

----

## UnixBench

VPS性能测试工具，耗时较长，耐心等待

```bash
curl https://codeload.github.com/kdlucas/byte-unixbench/zip/v5.1.3>UnixBench.zip
unzip UnixBench.zip
cd byte-unixbench-5.1.3/UnixBench
# apt-get install build-essential
make
screen -S ub
./Run
```

参考数据，均为最低配置：主机屋1590.5；阿里云1470.4；腾讯云1156.0

### 硬盘IO性能测试

```
dd if=/dev/zero of=test bs=64k count=4k oflag=dsync
dd if=/dev/zero of=test bs=8k count=256k conv=fdatasync
```

----

## 清除内存缓存

使用`free -h`查看可用内存前可以执行这个命令，查看除去buffer后的可用内存

```
sync
echo 3 > /proc/sys/vm/drop_caches
```

----

## 使用iptables封ip

### 屏蔽单个IP

    iptables -I INPUT -s 123.45.6.7 -j DROP

### 封C段

    iptables -I INPUT -s 123.45.6.0/24 -j DROP

### 封B段

     iptables -I INPUT -s 123.45.0.0/16 -j DROP

### 封A段

    iptables -I INPUT -s 123.0.0.0/8 -j DROP

记得**保存**：

    service iptables save

## 删除一条规则

只要把上述的插入规则重写一次，将其中的-I改为-D即可

    iptables -D INPUT -s IP地址 -j DROP

如果懒得重写 你也可以先列举出规则所在的id，根据id删除：

```
iptables -L --line-numbers
```

假设你想删除INPUT链的第3条规则：

```
iptables -D INPUT 3
```

## 只允许特定IP访问某端口

iptables的插入次序很重要，先加入的会先匹配，所以拒绝策略应该最后加入

以8888端口为例，只允许10.77.88.99这个IP 和 10.22.33.0/24 这个C段访问，其他来源的访问拒绝 返回connection refused

```
iptables -A INPUT -s 10.77.88.99 -p tcp --dport 8888 -j ACCEPT
iptables -A INPUT -s 10.22.33.0/24 -p tcp --dport 8888 -j ACCEPT
iptables -A INPUT -p tcp --dport 8888 -j REJECT
```

----

## 无root权限使用screen

> https://www.gnu.org/software/screen/

复制相同操作系统下的screen二进制文件，运行前指定环境变量

    mkdir -p $HOME/.screen
    export SCREENDIR=$HOME/.screen
    
----

## screen的用法

列出存在的screen：

    screen -ls
    
创建一个名为name的screen：

    screen -S name

从screen脱离：

    按Ctrl+A后按d

重新连上名称为name的screen：

    screen -r name

创建一个screen的自启动，让后台进程获得tty

    # 假设写好了一个/root/code.sh
    vim /etc/rc.local
    # 在最后加入一行，其中NAME替换为自己喜欢的名字
    screen -dmS NAME /root/code.sh

举个例子--监测外网能否ping通，如果不能重连zjuvpn：

[code/pingtest.sh](code/pingtest.sh)

----

## 双网卡端口转发，暴露内网端口

> 来自： https://yq.aliyun.com/wenzhang/show_25824

有两台机器，其中一台A 有内网和外网，B只有内网。

目标： 在外网访问A机器的2121端口，就相当于连上了B机器的ftp(21)

### 环境： 

A机器外网IP为 1.2.3.4(eth1) 内网IP为 192.168.1.20 (eth0)

B机器内网为 192.168.1.21

### 实现方法：

首先在A机器上打开端口转发功能

```
    echo 1 > /proc/sys/net/ipv4/ip_forward
    echo -e "\nnet.ipv4.ip_forward = 1">>/etc/sysctl.conf
    sysctl -p
```

然后在A机器上创建iptables规则

```
# 把访问外网2121端口的包转发到内网ftp服务器
iptables -t nat -I PREROUTING -d 1.2.3.4 -p tcp --dport 2121 -j DNAT --to 192.168.1.21:21 

# 把到内网ftp服务器的包回源到内网网卡上，不然包只能转到ftp服务器，而返回的包不能到达客户端
iptables -t nat -I POSTROUTING -d 192.168.1.21 -p tcp --dport 21 -j SNAT --to 192.168.1.20 

# 保存一下规则
service iptables save
```

### 取消转发方法

iptables中把-I改为-D运行就是删除此条规则

----

## 保护重要系统文件防止被删

使用+i标志位使得root用户也不能删除/bin, /sbin, /usr/sbin, /usr/bin, /usr/local/sbin, /usr/local/bin

```
chattr -R +i /bin /sbin /usr/sbin /usr/bin /usr/local/sbin /usr/local/bin
```

设置后无法apt-get安装新软件，需要先取消标志位

```
chattr -R -i /bin /sbin /usr/sbin /usr/bin /usr/local/sbin /usr/local/bin
```

----

## 时区设置

```
cp /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
ntpdate cn.pool.ntp.org
```

----

## 查看CPU核心个数

一般我会用 `top` 命令，按 `1` 就能看到每个CPU占用情况

但当CPU太多的时候还是需要执行命令的：

```
# 查看物理CPU个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l

# 查看每个物理CPU中core的个数(即核数)
cat /proc/cpuinfo| grep "cpu cores"| uniq

# 查看逻辑CPU的个数
cat /proc/cpuinfo| grep "processor"| wc -l
```

----

## 非交互式添加用户

```
useradd username -m
echo username:password|chpasswd
```

添加一个用户名为username的用户并创建home目录，并设置密码为password

----

## 简单OpenVPN配置

一个最最简单的场景：只有一个服务器 一个客户端，在容器中用来给用户直接访问的一个内网IP

参考：https://openvpn.net/index.php/open-source/documentation/miscellaneous/78-static-key-mini-howto.html

### 安装openvpn:

Linux:

```
apt-get install openvpn
```

Windows:

[openvpn.exe](https://d.py3.io/openvpn.exe)

### 生成密钥

```
openvpn --genkey --secret static.key
```

用另外建立的安全通道(SSH)将static.key发给客户端

### 服务端配置

```
ifconfig 10.8.0.1 10.8.0.2
secret /static.key
keepalive 10 60
persist-key
persist-tun
proto udp
port 1194
dev tun0
status /tmp/openvpn-status.log

user nobody
group nogroup
```

在 Ubuntu 中，如果要配置成系统服务的形式，将其保存到/etc/openvpn/myvpn.conf

然后这样启动它：

```
service openvpn@myvpn start
```

这样设置开机自启

```
systemctl enable openvpn@myvpn.service
```

### 客户端配置

```

remote 这里是你的服务器IP 这里是你的服务器端口 udp
dev tun
ifconfig 10.8.0.2 10.8.0.1
secret static.key
```

### 在Docker中使用服务端

参考：https://raw.githubusercontent.com/kylemanna/docker-openvpn/master/bin/ovpn_run

运行容器的时候一定要给参数`--cap-add=NET_ADMIN`

另外还需要在容器中执行：

```
mkdir -p /dev/net
if [ ! -c /dev/net/tun ]; then
    mknod /dev/net/tun c 10 200
fi
```

----

## 时区时间设置

参考：http://liumissyou.blog.51cto.com/4828343/1302050

```
apt-get install tzdata
cp /etc/localtime /etc/localtime.bak
ln -svf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
echo "TZ='Asia/Shanghai'">>~/.bashrc
```

修改时间可以用：

```
date -s "2017-06-18 16:40:00"
```

----

## 快速地格式化大分区ext4

Linux系统建议使用ext4分区格式，但直接mkfs.ext4 /dev/sda1就有很大的坑：会默认lazyinit在很长一段时间内占用IO

> 参考：[http://fibrevillage.com/storage/474-ext4-lazy-init](http://fibrevillage.com/storage/474-ext4-lazy-init)

正确格式化大硬盘的方法如下，这样不会跳过初始化磁盘的过程而且初始化过程很快：

```
mkfs.ext4 /dev/sdXX -E lazy_itable_init=0,lazy_journal_init=0 -O sparse_super,large_file -m 0 -T largefile4
```

对应的[man文档](http://manpages.ubuntu.com/manpages/precise/en/man8/mkfs.ext4.8.html)

----

## 添加受信任的CA证书 mitmproxy

```
echo ~/.mitmproxy/mitmproxy-ca-cert.pem >> /etc/ssl/certs/ca-certificates.crt
```

----

## 明明还有大量空间却说没有？inode满了！挂载单个文件为btrfs分区

### 问题现象

`df -h`显示还有很多空间，但`echo test>test.txt`会显示`No space left on device`

查到这个：https://wiki.gentoo.org/wiki/Knowledge_Base:No_space_left_on_device_while_there_is_plenty_of_space_available

使用`df -i`查看inodes占用情况，发现确实100%了

### 解决方案

inodes上限在mkfs时就定下来了，不能改动，所以没救了。。。

真没救了嘛？ 当然不是，虽然不能写入大量小文件，但还是可以写一个大文件的嘛，就想到挂载单个文件为btrfs分区

#### 1. 删文件 腾出部分inodes

删掉一些不用的小文件，也不用删太多

#### 2. 创建一个1TB的sparse file

参考: https://prefetch.net/blog/2009/07/05/creating-sparse-files-on-linux-hosts-with-dd/

使用dd命令，不将实际内容写入硬盘，能很快执行完成：

```
dd if=/dev/zero of=filesystem.img bs=1 count=0 seek=1T
```

执行后ls -alh能看到文件大小为1T，使用du filesystem.img查看真实空间

#### 3. 创建磁盘分区

参考：https://www.jamescoyle.net/how-to/2096-use-a-file-as-a-linux-block-device

btrfs参考：https://btrfs.wiki.kernel.org/index.php/Getting_started

```
mkfs.btrfs filesystem.img
```

#### 4. 挂载分区

首先找到一个空闲的loop设备：

```
losetup -f
```

假设得到了/dev/loop0，然后mount挂载咯：

```
sudo mount -o loop=/dev/loop0 /path/to/filesystem.img /mnt
```

#### 5. 然后就可以搬运数据过去了

就用mv像往常一样搬咯

#### 一些查看空间的命令

```
# 查看磁盘文件占用空间
du -h filesystem.img
# 查看btrfs元数据占用空间
btrfs filesystem df /mnt
# 也是查看空间
btrfs filesystem usage /mnt
```

#### 6. 卸载设备

```
sudo umount /mnt
sudo losetup -d /dev/loop0
```

----

## 安全地拔出移动硬盘

首先当然是`sudo umount /mnt`卸载挂载点咯，如何更安全一点呢？

```
udisksctl power-off -b /dev/sdb
```

From: https://unix.stackexchange.com/questions/354138/safest-way-to-remove-usb-drive-on-linux

----

## iptables 让监听在127.0.0.1上的端口可以公网访问

参考：https://unix.stackexchange.com/questions/111433/iptables-redirect-outside-requests-to-127-0-0-1

例如有监听在127.0.0.1:1234的应用，现在想通过ip:5678来访问

```
iptables -t nat -I PREROUTING -p tcp --dport 5678 -j DNAT --to-destination 127.0.0.1:1234
sysctl -w net.ipv4.conf.eth0.route_localnet=1
```