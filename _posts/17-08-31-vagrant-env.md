---
title: "Vagrant 统一开发环境"
layout: post
date: 2017-08-31 20:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
- Vagrant
- VirtualBox
category: blog
author: JamesiWorks
---

最近由于某种原因，需要在本地部署几个相同的开发环境；首先就想到了Vagrant，去官网一看，已经更新到1.9.8了，功能更加丰富，易用性更强，新增了Mirror查询服务，不用预先下载Box，添加Box更加简单。下面记录在OS X上部署统一开发环境时的一些关键步骤和遇到的问题，备忘。。。<br />
先预告一下，宿主机和客机共享文件夹是一个大坑；使用VirtualBox作为Provider时，默认的共享文件机制有Bug，会出现无法同步的问题。。。<br />

### install Vagrant & VirtualBox
```sh
    vagrant init
    vagrant box list
    vagrant box add centos/7
```

### Vagrantfile
```sh
    config.vm.box
    config.vm.box_version
    config.vm.box_url
    config.vm.network
    config.vm.synced_folder
```

### boot & rebuild
```sh
    vagrant up
    vagrant status
```

### reload & provision
```sh
    vagrant reload
    vagrant reload --provision
```

### teardown
```sh
    vagrant suspend
    vagrant halt
    vagrant destroy
```

### login in & out
```sh
    vagrant ssh
    ctrl + D
```

### synced folders & type
```sh
    ls /vagrant
```
#### RSync
从宿主机到客机，单向同步
```sh
    vagrant rsync
    vagrant rsync-auto
```
#### NFS
通过宿主机和客机的NFS服务器守护程序同步，双向同步
#### SMB
适用于Windows服务器，双向同步
#### VirtualBox
使用VirtualBox作为Provider时，默认使用VirtualBox共享文件夹，双向同步，但是有bug，WTF

### install apache
```sh
    sudo yum install httpd
    sudo service httpd start
    rpm -qa | grep httpd
    rpm -e xxx
```
可以通过宿主机浏览器验证 apache httpd 服务器是否正常启动；如 127.0.0.1:2333。

### vagrant providers
```sh
    vagrant up --provider=vmware_fusion
    vagrant up --provider=aws
```

### Perfect Vagrantfile
```sh
    config.vm.box = "centos/7"
    config.vm.box_version = "1707.01"
    config.vm.box_url = "https://vagrantcloud.com/centos/boxes/7/versions/1707.01/providers/virtualbox.box"
    config.vm.network "private_network", ip: "192.168.60.6"
    config.vm.network :forwarded_port, guest: 80, host:2333
    config.vm.synced_folder "shared", "/vagrant", type: "nfs"
```

### Done
That is all ...

