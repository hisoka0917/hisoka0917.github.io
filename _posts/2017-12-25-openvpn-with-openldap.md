---
layout: post
title:  OpenVPN使用OpenLDAP来做登录鉴权
date:   2017-12-25 16:03:00 +0800
categories: linux
tags: [linux, centos, openvpn, openldap]
---

在CentOS 7上OpenVPN的登录鉴权通过OpenLDAP来完成。有多种方法可以完成这个需求，比如使用openvpn的插件，比如使用pam。这里我们使用openvpn-auth-ldap插件来实现

## 安装openvpn-auth-ldap

从yum源安装`openvpn-auth-ldap`。OpenVPN和OpenLDAP的安装和配置在前2篇文章中已经讲述，这里不再重复了。

	yum install -y openvpn-auth-ldap
	
## 配置ldap.conf

备份原先的配置文件

	cp /etc/openvpn/auth/ldap.conf /etc/openvpn/auth/ldap.conf.bak
	
然后编辑`ldap.conf`

	# vim /etc/openvpn/auth/ldap.conf
	
	<LDAP>
        # LDAP server URL
        URL             ldap://127.0.0.1:389

        # Bind DN (If your LDAP server doesn't support anonymous binds)
        BindDN          cn=root,dc=example,dc=com

        # Bind Password
        Password        123456

        # Network timeout (in seconds)
        Timeout         15

        # Enable Start TLS
        TLSEnable       no

        # Follow LDAP Referrals (anonymously)
        FollowReferrals yes

        # TLS CA Certificate File
        TLSCACertFile   /usr/local/etc/ssl/ca.pem

        # TLS CA Certificate Directory
        TLSCACertDir    /etc/ssl/certs

        # Client Certificate and key
        # If TLS client authentication is required
        TLSCertFile     /usr/local/etc/ssl/client-cert.pem
        TLSKeyFile      /usr/local/etc/ssl/client-key.pem

        # Cipher Suite
        # The defaults are usually fine here
        # TLSCipherSuite        ALL:!ADH:@STRENGTH
	</LDAP>

	<Authorization>
        # Base DN
        BaseDN          "ou=People,dc=example,dc=com"

        # User Search Filter
        SearchFilter    "(&(uid=%u))"

        # Require Group Membership
        RequireGroup    false

        # Add non-group members to a PF table (disabled)
        #PFTable        ips_vpn_users

        <Group>
                BaseDN          "dc=example,dc=com"
                SearchFilter    "(|(cn=developers)(cn=artists))"
                MemberAttribute uniqueMember
                # Add group members to a PF table (disabled)
                #PFTable        ips_vpn_eng
        </Group>
	</Authorization>

填入你的LDAP服务器地址，然后绑定一个用户，建议新建一个专门的用户用来做这个事情。
TLS看你的需要开启。
然后填写认证的BaseDN和SearchFilter等项。如果你需要组来管理你的登录用户，开启`RequireGroup`然后配置你需要的组信息。

## 配置OpenVPN服务端

编辑`server.conf`文件，这个文件名看你安装的时候是配置的什么名字。添加如下2行代码

	# vim /etc/openvpn/server.conf
	
	plugin /usr/lib64/openvpn/plugin/lib/openvpn-auth-ldap.so "/etc/openvpn/auth/ldap.conf"
	client-cert-not-required

表示指定插件的位置和配置文件的位置，使用用户名密码登录而不使用证书。

## 配置客户端

在客户端配置文件中把使用证书的几项配置删除，修改成使用密码登录

	ca ca.crt
	;cert allen.crt
	;key allen.key
	auth-user-pass
	
重启OpenVPN服务器，就可以使用客户端连接了。

