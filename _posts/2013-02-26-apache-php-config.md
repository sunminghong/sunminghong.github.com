---
layout: post
categories: 
- work
- linux
title: centos6.2+Apache+php+mysql 配置备忘
description: 本文是我对一次不顺利的 centos6.2+Apache+php+mysql 配置做的备忘。
keywords: centos,apache,php,mysql
---

###1. 安装Apahce, PHP, Mysql, 以及php连接mysql库组件。
{% highlight bash lineno%}
su -l root #输入root密码切换到root

yum -y install httpd php mysql mysql-server php-mysql

/sbin/chkconfig httpd on [设置apache服务器httpd服务开机启动]
/sbin/service httpd start [启动httpd服务,与开机启动无关]create

/sbin/chkconfig --add mysqld [在服务清单中添加mysql服务]
/sbin/chkconfig mysqld on [设置mysql服务开机启动]
/sbin/service mysqld start [启动mysql服务,与开机无关]

{% endhighlight %}

###2.设置mysql数据库root帐号密码及基本配置
{% highlight bash lineno%}
mysqladmin -u root password 'newpassword' #[引号内填密码]

mysql -u root -p #[此时会要求你输入刚刚设置的密码，输入后回车即可]
mysql> DROP DATABASE test; #[删除test数据库]
mysql> DELETE FROM mysql.user WHERE user = ''; #[删除匿名帐户]
mysql> FLUSH PRIVILEGES; #[重载权限]

{% endhighlight %}

###3.关闭selinux安全系统
selinux 不关闭的情况下，我的网站总是不正常，总是提示“403 ”没有权限错误，所以特别记录如下.
vim /etc/selinux/config

在 SELINUX=enforcing 前面加个#号注释掉它
\#SELINUX=enforcing

然后新加一行
SELINUX=disabled

**保存，退出，重启系统，搞定。**

###4.httpd和php.ini配置略

###5.其它点滴记录：
+ 添加新用户 *grant all privileges on DATABASENAME.\* to 'USERNAME'@'localhost' identified by 'PASSWORD';


