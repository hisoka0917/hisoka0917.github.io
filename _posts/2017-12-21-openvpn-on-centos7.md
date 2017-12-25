---
layout: post
title:  在CentOS 7上用Firewalld配置OpenVPN
date:   2017-12-21 10:44:11 +0800
categories: linux
tags: linux
---

网上有很多配置OpenVPN的教程，但是大多数都是用的iptables。CentOS 7上默认使用Firewalld而不是iptables，这里我们尝试用Firewalld来配置。

### 安装OpenVPN和Easy RSA

从yum源安装openvpn和easy-rsa

    yum install -y openvpn easy-rsa
    
### 设置Easy RSA

新建一个文件夹来储存密钥和证书
	
	mkdir -p /etc/openvpn/easy-rsa/keys

将Easy RSA的脚本复制到OpenVPN的子目录下
	
	cp -R /usr/share/easy-rsa/2.0/ /etc/openvpn/easy-rsa/
	
编辑Easy RSA的配置文件

	vim /etc/openvpn/easy-rsa/2.0/vars
	
找到并修改这些值，填写你自己的信息。

    # These are the default values for fields
    # which will be placed in the certificate.
    # Don't leave any of these fields blank.
    export KEY_COUNTRY="US"
    export KEY_PROVINCE="CA"
    export KEY_CITY="SanFrancisco"
    export KEY_ORG="Fort-Funston"
    export KEY_EMAIL="me@myhost.mydomain"
    export KEY_OU="MyOrganizationalUnit"

然后找到这行
	
	export KEY_CONFIG=`$EASY_RSA/whichopensslcnf $EASY_RSA`
	
将之修改成
	
	export KEY_CONFIG=/etc/openvpn/easy-rsa/2.0/openssl-1.0.0.cnf
	
保存并退出

### 生成ca证书和密钥

输入以下命令初始化Easy RSA

	cd /etc/openvpn/easy-rsa/2.0
	chmod 0755 *
	source ./vars
	./clean-all

然后生成ca证书

	./build-ca
	
查看生成结果

    # ls -al keys
    total 20
    drwx------ 2 root root 4096 Jul 30 20:14 .
    drwxr-xr-x 3 root root 4096 Jul 30 20:09 ..
    -rw-r--r-- 1 root root 1887 Jul 30 20:14 ca.crt
    -rw------- 1 root root 1704 Jul 30 20:14 ca.key
    -rw-r--r-- 1 root root 0 Jul 30 20:09 index.txt
    -rw-r--r-- 1 root root 3 Jul 30 20:09 serial

### 生成vpn客户端证书和私钥

	./build-key-server server

注意这行留空

	A challenge password []: <= 这行留空

查看keys文件夹

	ls -al keys

为客户端的用户生成证书

	./build-key allen
	
这里要填一个challenge password:

	A challenge password []: ChooseASafePassword123

### 生成Diffie Hellman交换文件

键入以下命令生成.pem文件

	./build-dh

### 拷贝证书和密钥

	cd /etc/openvpn/easy-rsa/2.0/keys
	cp dh2048.pem ca.crt server.crt server.key /etc/openvpn

### 生成OpenVPN配置文件

拷贝一个示例配置文件

	cp /usr/share/doc/openvpn-2.4.4/sample/sample-config-files/server.conf /etc/openvpn/server.conf
 	
确保用户"nobody"能读这些文件

	cd /etc/openvpn
	chmod 0644 dh2048.pem ca.crt server.crt server.key server.conf
  
编辑配置文件

	vim /etc/openvpn/server.conf

找到以下这行并取消注释
	
	push "redirect-gateway def1 bypass-dhcp"
	
找到下面这行
		
	dev tun

添加如下设置

	dev tun
	tun-mtu 1500
	tun-mtu-extra 32
	mssfix 1450
	reneg-sec 0
	
把下面这几行注释去掉

	; push "dhcp-option DNS 208.67.222.222"
	; push "dhcp-option DNS 208.67.220.220"

并改成你自己的dns

	push "dhcp-option DNS 8.8.8.8"
	push "dhcp-option DNS 8.8.4.4"

去掉以下几行的注释

	user nobody
	group nobody

开启压缩

	comp-lzo

关闭退出通知
	
	explicit-exit-notify 0

保存并退出

### 开启ip转发和路由

编辑/etc/sysctl.conf然后添加参数
	
	net.ipv4.ip_forward = 1
	
保存退出并重启服务使配置生效
	
	systemctl restart network.service

现在我们需要配置Firewalld。让OpenVPN服务能通过防火墙并让配置永久生效。

	firewall-cmd --add-service openvpn
	firewall-cmd --permanent --add-service openvpn

查看服务添加是否成功

	firewall-cmd --list-services

类似的输出如下
	
	ssh dhcpv6-client openvpn

启用masquerade并使之永久化

	firewall-cmd --add-masquerade
	firewall-cmd --permanent --add-masquerade

验证masquerade已启用
    
    # firewall-cmd --query-masquerade
    yes

### 启动VPN服务

把OpenVPN添加到服务并设置开机启动

	systemctl -f enable openvpn@server.service

启动服务

	systemctl start openvpn@server.service

如果启动不成功，用`systemctl status`查看信息，如果显示缺少ta.key文件，则可以用以下命令创建
	
	openvpn –genkey –secret ta.key

如果不需要防范DoS攻击或者不需要很高的安全性，可以在服务器配置文件中将`tls-auth ta.key 0`这行去掉。

接下来就可以用OpenVPN客户端来连接服务了。

### 配置客户端

从服务器上下载这几个文件：

-	/etc/openvpn/ca.crt
-	/etc/openvpn/easy-rsa/2.0/keys/allen.crt
-	/etc/openvpn/easy-rsa/2.0/keys/allen.csr
-	/etc/openvpn/easy-rsa/2.0/keys/allen.key
-	/usr/share/doc/openvpn-2.4.4/sample/sample-config-files/client.conf

用文本编辑器打开`client.conf`，修改以下内容

	remote my-server-1 1194  //修改成自己的服务器地址
	ca ca.crt
	cert allen.crt
	key allen.key
	tun-mtu 1500
	tun-mtu-extra 32
	mssfix 1450
	reneg-sec 0
	comp-lzo

保存文件。
然后将该文件导入到OpenVPN客户端，就可以连接服务器了。

	


