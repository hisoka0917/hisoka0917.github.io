---
layout: post
title:  OpenLDAP用户和组的类型
date:   2017-12-26 16:50:00 +0800
categories: linux
tags: [linux, openldap]
---

前几篇文章讲了OpenLDAP的一些安装配置。在使用OpenLDAP的过程当中也是踩了一些坑，主要是LDAP用户的类型。

我们使用OpenVPN的时候使用OpenLDAP来进行鉴权，根据前文的配置，基本是可以使用的。不过我们希望把那些可以使用vpn的用户加到一个组里，这样方便管理。于是我们建了一个`cn=vpn`的组，然后编辑`memberUid`把某几个用户加进来。然后编辑`ldap.conf`

	<Authorization>
		# Base DN
		BaseDN		"ou=People,dc=example,dc=com"

		# User Search Filter
		SearchFilter	"(uid=%u)"

		# Require Group Membership
		RequireGroup	true

		# Add non-group members to a PF table (disabled)
		#PFTable	ips_vpn_users

		<Group>
			BaseDN		"ou=Group,dc=example,dc=com"
			SearchFilter	"(cn=vpn)"
			MemberAttribute	memberUid
			# Add group members to a PF table (disabled)
			#PFTable	ips_vpn_eng
		</Group>
	</Authorization>

完成配置后重启服务器，尝试使用已加到这个组里的用户登录，结果返回结果验证失败。

通过几次尝试以及使用不同的ldap客户端终于找到了原因。我们之前创建用户和组都是使用的`phpldapadmin`，并且创建的时候选择了`Generic: User Account`，创建组的时候选择了`Generic: Posix Group`。这样创建出来的用户的`objectClass`值是`top, posixAccount, inetOrgPerson`，创建的组的`objectClass`是`top, posixGroup`，组里面关联用户的key是`memberUid`。

使用别的ldap客户端创建的用户`objectClass`是`top, person, organizationalPerson, inetOrgPerson`，组的`objectClass`是`top, groupOfNames`，以及组里面关联用户的key是`member`。当我们使用别的客户端新建了`groupOfNames`类型的组，并且把原来的用户加到`member`属性下。在ldap.conf文件中也把组相关的配置改成`MemberAttribute	member`。再次尝试连接成功。

这里面可能是和`openvpn-auth-ldap`这个插件有关，在现在这种情况下只能这种类型的组才能完成验证。在phpldapadmin里面这种组叫做`Samba: Group Mapping`。

由于之前一直是使用phpldapadmin所以没在意这种类型为题。这提醒我要好好组织一下ldap的目录结构以及每个节点的类型。objectClass的类型在和Windows AD做关联的时候也许也是很重要的。


