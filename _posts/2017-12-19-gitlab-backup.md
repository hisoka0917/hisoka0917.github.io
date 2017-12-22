---
layout: post
title:  "Gitlab自动备份"
date:   2017-12-19 16:50:11 +0800
categories: git
tags: git
---

近日由于需要做了一下gitlab的自动备份。总的方法是挂载一个共享目录，将gitlab的备份目录改到该共享目录，并使用crontab来实现每日自动备份。

## 挂载共享目录

目前的情况是这样的，服务器是在一台Windows Server上，用Hyper-V创建的Linux虚拟机，gitlab搭建在虚拟机里。Windows Server上有一块硬盘作为NAS。我的想法是在NAS上创建一个共享目录，然后把这个目录挂载到Linux上。gitlab支持NFS/CIFS/SMB等协议的共享目录备份。
首先在Windows上设置好共享目录，然后分配好用户和访问权限。然后到Linux里面编辑`/etc/fstab`文件，添加以下一行：
    
    \\192.168.1.62\share-dir /mnt/backups cifs credentials=/root/secret.txt,uid=git,gid=git 0 0
    
`secret.txt`的内容为拥有共享目录访问权限的用户名和密码，例如：

    username=gitlab
    password=123456
    
这行代码的意思是使用cifs协议，将`\\192.168.1.62\share-dir`文件夹挂载到`/mnt/backups`上。访问该路径的用户信息在`secret.txt`里面，并且赋予git用户的权限。

保存以后运行`mount -a`挂载目录。写在`/etc/fstab`里的好处是每次开机重启后会自动挂载。

## 修改gitlab备份路径

编辑`/etc/gitlab/gitlab.rb`文件，添加如下内容：

{% highlight ruby %}
gitlab_rails['backup_upload_connection'] = {
  :provider => 'Local',
  :local_root => '/mnt/backups'
}

# The directory inside the mounted folder to copy backups to
# Use '.' to store them in the root directory
gitlab_rails['backup_upload_remote_directory'] = 'gitlab_backups'
{% endhighlight %}

gitlab生成的备份文件访问权限为owner/group是git:git并且访问权限是0600，如果想要修改生成备份文件的访问权限，就添加以下内容：

{% highlight ruby %}
# In /etc/gitlab/gitlab.rb, for omnibus packages
gitlab_rails['backup_archive_permissions'] = 0644 # Makes the backup archives world-readable
{% endhighlight %}

如果想要让gitlab自动清理旧的备份，添加如下内容：

{% highlight ruby %}
# limit backup lifetime to 7 days - 604800 seconds
gitlab_rails['backup_keep_time'] = 604800
{% endhighlight %}

编辑完成后执行`gitlab-ctl reconfigure`让配置生效。

## 使用crontab执行每日自动备份

执行`sudo crontab -e -u root`编辑crontab。

    0 3 * * 2-6  umask 0077; tar cfz /home/backups/gitlab-backups/$(date "+etc-gitlab-\%s.tgz") -C / etc/gitlab
    0 4 * * * /opt/gitlab/bin/gitlab-rake gitlab:backup:create CRON=1

第一行是说在每周2到周6的3点钟备份`/etc/gitlab`目录。该目录下存放着gitlab的配置文件，用户的二次验证信息等。gitlab官方推荐备份整个文件夹。

第二行意思是在每天4点使用gitlab本身的备份命令备份整个应用。`CRON=1`表示如果没有错误就不输出进度，这样会减少cron的垃圾信息，推荐加上该参数。

**注意：我们挂载了共享文件夹到`/mnt/backups/`路径下，上述crontab的tar命令往这个路径上创建压缩文件会失败。所以要找另外一个路径来备份。**

