---
layout: post
title: 使用git hooks script（钩子脚本）进行版本自动发布 - git备忘录
description: 本文记录了孙铭鸿在实际的游戏开发中，通过git 分支或标签自动发不多个版本的实现过程及相关脚本。
keywords: git,hooks,script,branch,tag,钩子,脚本,分支,标签,游戏开发，备忘录
---
###干货备忘录
######本文的适应环境及声明
* 发布文件夹所在的服务器上有git版本库
* 否则，只有钩子脚本可供参考
* 我不是git专家，我对git的使用观念是够用就好，本文是满足我需求的实践总结，大家随意交流，如有转载是我的荣幸！。

本文模拟一个场景：我们需要在服务器 **SERVER** 上将 ***/home/git/repositories/sg/xxx.git*** 提交自动发布/更新到**/home/game/**下同名的文件夹下，例如的 **branchA** 分支自动发布/更新到 ***/home/game/branchA*** ，**branchB** 分支自动发布/更新到 ***/home/game/branchB***。
\****/home/game***下没有相应的分支文件夹的就不自动发布。
\*git 服务器的帐号**git**

######1、建立发布文件夹
这一步超重要，这是经过几天折腾“升华”出的最佳流程，如果读者想尽快投入使用，照着做是做就可以了。

{% highlight bash lineno%}
$su -l git #切换到git帐号，适用于给git服务器程序专门的帐号的情况（推荐这样部署git服务器）
$umask 002
$cd /home/game
$git clone /home/git/repositories/sg/xxx.git -b branchA branchA
$git clone /home/git/repositories/sg/xxx.git -b branchB branchB
{% endhighlight %}

######2、git钩子脚本

{% highlight bash lineno%}
$cd /home/git/repositories/sg/sgserver.git/hooks
$vim post-receive #写入如下代码，保存
$chmod +x post-receive
{% endhighlight %}

{% highlight python lineno%}
#!/usr/bin/python
#coding:utf-8
#hoos/post-receive

import sys,traceback
import os,time
import subprocess
from datetime import datetime

def git(args):
    args = ['git'] + args
    git = subprocess.Popen(args, stdout = subprocess.PIPE)
    details = git.stdout.read()
    details = details.strip()
    return details

def get_repo_name():
    if git(['rev-parse','--is-bare-repository']) == 'true':
        name = os.path.basename(os.getcwd())
        if name.endswith('.git'):
            name = name[:-4]
        return name
    else:
        return os.path.basename(os.path.dirname(os.getcwd()))

REPO_NAME = get_repo_name()

if __name__ == '__main__':
    for line in sys.stdin.xreadlines():
        old, new, ref = line.strip().split(' ')

	branchname = ref.replace('refs/heads/','')
	sys.stdout.write("\nallen's hooks receive %s,%s\n" % (REPO_NAME,branchname))

	# 该分支的发布目录不存在就退出
	pullpath = "/opt/%s" % (branchname)
	if not os.path.exists(pullpath):
		#os.system('mkdir '+ pullpath) 
		exit(0)	

	nowtime = time.strftime('%Y-%m-%d %H:%M:%s')
	f=file(r'/var/log/tmp/git-post-receive-%s.log' % REPO_NAME,'aw')
	f.write('%s----%s-----\n' % (ref,nowtime))
	try:
		cmd = 'cd '+ pullpath +'; env -i git log -1 --pretty=format:"%s"'
		logcomment = os.popen(cmd).read()

		f.write('logcomment:%s\n' % logcomment)
		f.write('%s	%s\n' % (old , new ))
		f.write('receive %s,%s\n' % (REPO_NAME,branchname))

		cmd = 'umask 002; cd %s/;env -i git pull' % (pullpath)
		f.write('cmd:%s\n' % (cmd))
		os.system(cmd)
		f.close()

		sys.stdout.write(',pull complete!\n')
	except:
		msg = traceback.sys_exc()
		sys.stderr.write(msg)
		f.close()
{% endhighlight %}

