---
layout: post
title: shadowsocks-libv使用ss-manager实现多用户使用
date: 2018-1-19
categories: shadowsocks
tags: [shadowsocks, shadowsocks-libv, ss-manager, centos, linux]
---

shadowsocks目前只有libv版本还在持续更新并支持AEAD加密算法。但是libv版本本身不支持多用户使用。不过ss-manager可以实现这一目的。以下使用CentOS 7作为例子来实现多用户使用。

1. 安装shadowsocks-libv，这一步的教程很多，就不重复了。

2. 编辑配置文件

```bash
$vim /etc/shadowsocks/config.json

{
  "server":["::0","0.0.0.0"],
  "port_password": {
        "8848": "password1",
        "8838": "password2"
  },
  "timeout":60,
  "method":"chacha20-ietf-poly1305",
  "fast_open":true
}

```

配置文件写法和以前的多端口写法相同。

3. 创建系统服务

```bash
$vim /etc/systemd/system/ssmanager.service

[Unit]
Description=Shadowsocks Manager Server

[Service]
User=root
Group=root
Type=simple
ExecStart=/usr/local/bin/ss-manager --manager-address /var/run/shadowsocks-manager.sock -c /etc/shadowsocks/config.json -a shadowsocks start
ExecStop=/usr/local/bin/ss-manager --manager-address /var/run/shadowsocks-manager.sock -c /etc/shadowsocks/config.json -a shadowsocks stop

[Install]
WantedBy=multi-user.target
```

保存文件。

4. 设置开机启动

```bash
systemctl enable ssmanager
systemctl start ssmanager
```

配置好防火墙就可以使用了，而且之前配置的shadowsocks服务可以停用了。