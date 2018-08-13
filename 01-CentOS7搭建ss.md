
# CentOS7搭建ss

>引用自 http://blog.51cto.com/zero01/2064660

作为一个新世纪的码农，我们经常需要使用百度以及Google等搜索引擎搜索资料或搜索一些错误的解决方案，如果English好的还可能需要到stackoverflow里查看或提问一些开发中遇到的问题，再者可能还需要到youtube上查找一些教学、科普视频等等。还好的是stackoverflow部分不牵扯Google的内容在国内还是能够正常访问的，但是Google和youtube嘛大家都懂，所以本文就介绍一下如何在vps上搭建shadowsocks，让我们能够访问这些网站，以便于我们查阅资料，切勿用做其他不法用途。

常见VPS的goumai地址 :
活跃于大街小巷的搬瓦工，也是最适合新手使用的：

https://bwh1.net/ （支持支付宝）

我目前使用的vultr，以下是我的分享链接：

https://www.vultr.com/?ref=7315390（支持支付宝）

SugarHosts：

https://www.sugarhosts.com/zh-cn/

Linode：

https://www.linode.com/

Virmach：

https://billing.virmach.com/cart.php?gid=1（支持支付宝）

RAKSmart：

https://billing.raksmart.com/

Bluehost：

https://cn.bluehost.com/

DigitalOcean：

https://www.digitalocean.com

以上这些都是国外的vps，国内的可以购买阿里云或者腾讯云等，国内没有遇到优惠的话就比较贵。ps：我在想要不要问他们给广告费2333。

安装 pip
pip是 python 的包管理工具。在本文中将使用 python 版本的 shadowsocks，此版本的 shadowsocks 已发布到 pip 上，因此我们需要通过 pip 命令来安装。

在控制台执行以下命令安装 pip：
[root@server ~]# curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
[root@server ~]# python get-pip.py

安装配置 shadowsocks
在控制台执行以下命令安装 shadowsocks：

[root@server ~]# pip install --upgrade pip
[root@server ~]# pip install shadowsocks
安装完成后，需要创建shadowsocks的配置文件/etc/shadowsocks.json，编辑内容如下：

[root@server ~]# vim /etc/shadowsocks.json
{
  "server": "0.0.0.0",
  "local_address": "127.0.0.1",
  "local_port": 1080,
  "port_password": {
    "8080": "填写密码",
    "8081": "填写密码"
  },
  "timeout": 600,
  "method": "aes-256-cfb"
}
说明：

method为加密方法，可选aes-128-cfb, aes-192-cfb, aes-256-cfb, bf-cfb, cast5-cfb, des-cfb, rc4-md5, chacha20, salsa20, rc4, table
port_password为端口对应的密码，可使用密码生成工具生成一个随机密码
以上两项信息在配置 shadowsocks 客户端时需要配置一致，具体说明可查看 shadowsocks 的帮助文档。

如果你不需要配置多个端口的话，仅配置单个端口，则可以使用以下配置：

{
  "server": "0.0.0.0",
  "server_port": 8080,
  "password": "填写密码",
  "method": "aes-256-cfb"
}
说明：

server_port为服务监听端口
password为密码
同样的以上两项信息在配置 shadowsocks 客户端时需要配置一致。

配置自启动
编辑shadowsocks 服务的启动脚本文件，内容如下：

[root@server ~]# vim /etc/systemd/system/shadowsocks.service
[Unit]
Description=Shadowsocks

[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/ssserver -c /etc/shadowsocks.json

[Install]
WantedBy=multi-user.target
执行以下命令启动 shadowsocks 服务：

[root@server ~]# systemctl enable shadowsocks
[root@server ~]# systemctl start shadowsocks
检查 shadowsocks 服务是否已成功启动，可以执行以下命令查看服务的状态：

systemctl status shadowsocks -l

如果服务启动成功，则控制台显示的信息应该类似这样：


CentOS7.4搭建shadowsocks，以及配置BBR加速
确认服务启动成功后，配置防火墙规则，开放你配置的端口，不然客户端是无法连接的：

[root@server ~]# firewall-cmd --zone=public --add-port=8080/tcp --permanent
success
[root@server ~]# firewall-cmd --zone=public --add-port=8081/tcp --permanent
success
[root@server ~]# firewall-cmd --reload
success
附上一键安装脚本代码：

#!/bin/bash
# Install Shadowsocks on CentOS 7

echo "Installing Shadowsocks..."

random-string()
{
    cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w ${1:-32} | head -n 1
}

CONFIG_FILE=/etc/shadowsocks.json
SERVICE_FILE=/etc/systemd/system/shadowsocks.service
SS_PASSWORD=$(random-string 32)
SS_PORT=8388
SS_METHOD=aes-256-cfb
SS_IP=`ip route get 1 | awk '{print $NF;exit}'`
GET_PIP_FILE=/tmp/get-pip.py

# install pip
curl "https://bootstrap.pypa.io/get-pip.py" -o "${GET_PIP_FILE}"
python ${GET_PIP_FILE}

# install shadowsocks
pip install --upgrade pip
pip install shadowsocks

# create shadowsocls config
cat <<EOF | sudo tee ${CONFIG_FILE}
{
  "server": "0.0.0.0",
  "server_port": ${SS_PORT},
  "password": "${SS_PASSWORD}",
  "method": "${SS_METHOD}"
}
EOF

# create service
cat <<EOF | sudo tee ${SERVICE_FILE}
[Unit]
Description=Shadowsocks

[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/ssserver -c ${CONFIG_FILE}

[Install]
WantedBy=multi-user.target
EOF

# start service
systemctl enable shadowsocks
systemctl start shadowsocks

# view service status
sleep 5
systemctl status shadowsocks -l

echo "================================"
echo ""
echo "Congratulations! Shadowsocks has been installed on your system."
echo "You shadowsocks connection info:"
echo "--------------------------------"
echo "server:      ${SS_IP}"
echo "server_port: ${SS_PORT}"
echo "password:    ${SS_PASSWORD}"
echo "method:      ${SS_METHOD}"
echo "--------------------------------"
配置客户端
我这里配置的是windows的客户端，挺方便的，点击即用，不需要安装。

Windows客户端下载地址：

https://github.com/shadowsocks/shadowsocks-windows/releases

Mac客户端下载地址：

https://github.com/shadowsocks/ShadowsocksX-NG/releases

Android客户端下载地址：

https://github.com/shadowsocks/shadowsocks-android/releases

运行客户端程序，右键点击shadowsocks图标，然后点击编辑服务器：


CentOS7.4搭建shadowsocks，以及配置BBR加速
配置对应的信息：


CentOS7.4搭建shadowsocks，以及配置BBR加速
CentOS7.4搭建shadowsocks，以及配置BBR加速
然后显示已启用代表配置成功：


CentOS7.4搭建shadowsocks，以及配置BBR加速
接着测试能否上Google搜索即可，以下的配置BBR加速则是选看，不配置也是可以正常使用shadowsocks的。

配置BBR加速
什么是BBR：

TCP BBR是谷歌出品的TCP拥塞控制算法。BBR目的是要尽量跑满带宽，并且尽量不要有排队的情况。BBR可以起到单边加速TCP连接的效果。

Google提交到Linux主线并发表在ACM queue期刊上的TCP-BBR拥塞控制算法。继承了Google“先在生产环境上部署，再开源和发论文”的研究传统。TCP-BBR已经再YouTube服务器和Google跨数据中心的内部广域网(B4)上部署。由此可见出该算法的前途。

TCP-BBR的目标就是最大化利用网络上瓶颈链路的带宽。一条网络链路就像一条水管，要想最大化利用这条水管，最好的办法就是给这跟水管灌满水。

BBR解决了两个问题：

在有一定丢包率的网络链路上充分利用带宽。非常适合高延迟，高带宽的网络链路。

降低网络链路上的buffer占用率，从而降低延迟。非常适合慢速接入网络的用户。

Google 在 2016年9月份开源了他们的优化网络拥堵算法BBR，最新版本的 Linux内核(4.9-rc8)中已经集成了该算法。

对于TCP单边加速，并非所有人都很熟悉，不过有另外一个大名鼎鼎的商业软件“锐速”，相信很多人都清楚。特别是对于使用国外服务器或者VPS的人来说，效果更佳。

BBR项目地址：

https://github.com/google/bbr

升级内核，第一步首先是升级内核到支持BBR的版本：
1.yum更新系统版本：

yum update

2.查看系统版本：

[root@server ~]# cat /etc/redhat-release 
CentOS Linux release 7.4.1708 (Core) 
[root@server ~]# 
3.安装elrepo并升级内核：

[root@server ~]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
[root@server ~]# rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
[root@server ~]# yum --enablerepo=elrepo-kernel install kernel-ml -y
4.更新grub文件并重启系统：

[root@server ~]# egrep ^menuentry /etc/grub2.cfg | cut -f 2 -d \'
CentOS Linux 7 Rescue 8619ff5e1306499eac41c02d3b23868e (4.14.14-1.el7.elrepo.x86_64)
CentOS Linux (4.14.14-1.el7.elrepo.x86_64) 7 (Core)
CentOS Linux (3.10.0-693.11.6.el7.x86_64) 7 (Core)
CentOS Linux (3.10.0-693.el7.x86_64) 7 (Core)
CentOS Linux (0-rescue-c73a5ccf3b8145c3a675b64c4c3ab1d4) 7 (Core)
[root@server ~]# grub2-set-default 0
[root@server ~]# reboot
5.重启完成后查看内核是否已更换为4.14版本：

[root@server ~]# uname -r
4.14.14-1.el7.elrepo.x86_64
[root@server ~]#
6.开启bbr：

[root@server ~]# vim /etc/sysctl.conf    # 在文件末尾添加如下内容
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
7.加载系统参数：

[root@vultr ~]# sysctl -p
net.ipv6.conf.all.accept_ra = 2
net.ipv6.conf.eth0.accept_ra = 2
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
[root@vultr ~]#
如上，输出了我们添加的那两行配置代表正常。

8.确定bbr已经成功开启：

[root@vultr ~]# sysctl net.ipv4.tcp_available_congestion_control
net.ipv4.tcp_available_congestion_control = bbr cubic reno
[root@vultr ~]# lsmod | grep bbr
tcp_bbr                20480  2 
[root@vultr ~]# 
输出内容如上，则表示bbr已经成功开启。
