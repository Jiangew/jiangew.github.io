---
title: "Scientific Surfing the Internet"
layout: post
date: 2018-03-15 20:30
image: /assets/images/base/markdown.jpg
headerImage: false
tag:
- VPS
- BBR
- IPTABLES
- Shadowsocks
category: blog
author: jiangew
---

- [CentOS 7 Deploy Google BBR](#centos-7-deploy-google-bbr)
  - [Preparation](#preparation)
  - [Enable BBR](#enable-bbr)
- [CentOS 7 Deploy Shadowsocks](#centos-7-deploy-shadowsocks)
  - [Install pip](#install-pip)
  - [Install and Config Shadowsocks](#install-and-config-shadowsocks)
  - [Enable Launch Service](#enable-launch-service)
  - [ALL-IN-ONE Script](#all-in-one-script)
- [Shadowsocks Limit Device Connections and Port Speeds](#shadowsocks-limit-device-connections-and-port-speeds)
  - [Install IPTABLES](#install-iptables)
  - [IPTABLES 限定端口连接数](#iptables-%E9%99%90%E5%AE%9A%E7%AB%AF%E5%8F%A3%E8%BF%9E%E6%8E%A5%E6%95%B0)
  - [IPTABLES 限制端口速度](#iptables-%E9%99%90%E5%88%B6%E7%AB%AF%E5%8F%A3%E9%80%9F%E5%BA%A6)
  - [IPTABLES 限制IP访问速度](#iptables-%E9%99%90%E5%88%B6ip%E8%AE%BF%E9%97%AE%E9%80%9F%E5%BA%A6)
  - [Shadowsocks 限制设备连接数](#shadowsocks-%E9%99%90%E5%88%B6%E8%AE%BE%E5%A4%87%E8%BF%9E%E6%8E%A5%E6%95%B0)

## CentOS 7 Deploy Google BBR
BBR (Bottleneck Bandwidth and RTT) 是由 Google 贡献给 Linux 内核 `TCP` 协议栈的一种新的拥塞控制算法。通过使用 `BBR` 可以显著提高 Linux 服务器的吞吐量，并减少连接延迟。此外，BBR 部署也很简单，因为此算法只需在发送端更新部署即可，而无需在网络或者接收端更新部署。

原文地址：[How to Deploy Google BBR on CentOS 7](https://www.vultr.com/docs/how-to-deploy-google-bbr-on-centos-7)

### Preparation
进入`ssh`后不是`root`权限，先获取`root`权限：
```sh
sudo -i
```

查看当前内核版本，确保版本高于 `4.9`；如果低，自行升级内核；
```sh
uname –a
```

### Enable BBR
写入配置：
```sh
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
```

使配置生效：
```sh
sysctl -p
```

检验是否生效：
```sh
lsmod | grep bbr
```

看到显示 `tcp_bbr 20480 0` 说明已经成功开启 `BBR`，不需要重新启动。

## CentOS 7 Deploy Shadowsocks
CentOS 7 开始默认使用 Systemd 作为开启启动脚本的管理工具，[Shadowsocks](https://github.com/shadowsocks/) 则是当前比较受欢迎的科学上网工具，本文将介绍如何在 CentOS 下安装和配置 `Shadowsocks` 服务。

### Install pip
`pip` 是 python 的包管理工具。在本文中将使用 python 版本的 shadowsocks，此版本的 shadowsocks 已发布到 pip 上，因此我们需要通过 pip 命令来安装。

在控制台执行以下命令安装 pip:
```sh
curl "https://bootstrap.pypa.io/get-pip.py" -o "get-pip.py"
python get-pip.py
```

### Install and Config Shadowsocks
```sh
pip install --upgrade pip
pip install shadowsocks
```

安装完成后，需要创建配置文件 `/etc/shadowsocks.json`，内容如下：
```json
{
  "server": "0.0.0.0",
  "server_port": 8080,
  "password": "a123456789BC",
  "method": "aes-256-cfb"
}
```
以上三项信息在配置 shadowsocks 客户端时需要配置一致，具体说明可查看 shadowsocks 的帮助文档。

### Enable Launch Service
新建启动脚本文件 `/etc/systemd/system/shadowsocks.service`，内容如下：
```config
[Unit]
Description=Shadowsocks

[Service]
TimeoutStartSec=0
ExecStart=/usr/bin/ssserver -c /etc/shadowsocks.json

[Install]
WantedBy=multi-user.target
```

执行以下命令启动 shadowsocks 服务：
```sh
systemctl enable shadowsocks
systemctl start shadowsocks
```

为了检查 shadowsocks 服务是否已成功启动，可以执行以下命令查看服务的状态：
```sh
systemctl status shadowsocks -l
```

### ALL-IN-ONE Script
新建文件 `install-shadowsocks.sh`，内容如下：
```sh
#!/bin/bash
# Install Shadowsocks on CentOS 7

echo "Installing Shadowsocks ..."

random-string()
{
    cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w ${1:-32} | head -n 1
}

CONFIG_FILE=/etc/shadowsocks.json
SERVICE_FILE=/etc/systemd/system/shadowsocks.service
SS_PASSWORD=$(random-string 32)
SS_PORT=8080
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
```

执行以下命令一键安装：
```sh
chmod +x install-shadowsocks.sh
./install-shadowsocks.sh
```

安装完成后会自动打印出 `Shadowsocks` 的连接配置信息。比如：
```json
Congratulations! Shadowsocks has been installed on your system.
You shadowsocks connection info:
--------------------------------
server:      0.0.0.0
server_port: 8080
password:    a123456789BC
method:      aes-256-cfb
--------------------------------
```

## Shadowsocks Limit Device Connections and Port Speeds
如何使自己搭建的 `Shadowsocks` 服务能够 `限制设备连接数` 和 `限制端口速度` 呢？

### Install IPTABLES
```sh
yum install iptables-services
systemctl enable iptables
systemctl [stop|start|restart] iptables
```

### IPTABLES 限定端口连接数
首先输入命令 `service iptables stop` 关闭 `iptables`；限制端口并发数很简单，`iptables` 就能搞定了，假设你要限制端口`8080`的`IP`最大连接数为`10`，两行命令搞定：
```sh
iptables -I INPUT -p tcp --dport 8080 -m connlimit --connlimit-above 10 -j DROP
iptables -I OUTPUT -p tcp --dport 8080 -m connlimit --connlimit-above 10 -j DROP
```

如果你想限制从 `1024-10240` 的端口：
```sh
iptables -I INPUT -p tcp --dport 1024:10240 -m connlimit --connlimit-above 10 -j DROP
iptables -I OUTPUT -p tcp --dport 1024:10240 -m connlimit --connlimit-above 10 -j DROP
```

保存 `iptables` 规则：
```sh
service iptables save
```

启动 `iptables`：
```sh
service iptables start
```

查看是否生效：
```sh
iptables -L -n -v
```

### IPTABLES 限制端口速度
输入命令 `service iptables stop` 关闭 `iptables`；假设你要限制端口`5037`的最大连接速度为`60`个包每秒，两行命令搞定：
```sh
iptables -A INPUT -p tcp --sport 5037 -m limit --limit 60/s -j ACCEPT
iptables -A INPUT -p tcp --sport 5037 -j DROP
```
也就是限制每秒接受`60`个包，一般来说每个包大小为`64—1518`字节(Byte)。

### IPTABLES 限制IP访问速度
原理是每秒对特定端口进行速度控制，比如每秒超过`10`个的数据包直接`DROP`，从而限制特定端口的速度。
```sh
iptables -A FORWARD -m limit -d 208.8.14.53 --limit 700/s --limit-burst 100 -j ACCEPT 
iptables -A FORWARD -d 208.8.14.53 -j DROP
```

最后说一下如何解决防火墙重启后失败的问题：
```sh
iptables-save > /etc/sysconfig/iptables
echo 'iptables-restore /etc/sysconfig/iptables' >> /etc/rc.local
chmod +x /etc/rc.d/rc.local
```

### Shadowsocks 限制设备连接数
打开你的 `Shadowsocks` 服务端配置文件：
```sh
vi /etc/shadowsocks.json
```

找到协议参数（参数为空 "" 时，默认限制`64`个设备数）：
```json
"protocol_param": "",
```

在协议参数中设置你要限制每个端口最大设备连接数（建议`最少2个`），比如限制`最大10`个设备同时链接：
```json
"protocol_param": "10",
```

OK，到这里你的科学上网工具就搭建完毕了，赶快自由的冲浪去吧 ~
