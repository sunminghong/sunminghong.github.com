---
layout: post
title: python写git钩子脚本实现多个版本的自动发布 - git备忘录
description: 本文记录了孙铭鸿在实际的游戏开发中，通过git 分支或标签自动发不多个版本的实现过程及相关脚本。
keywords: git,hooks,script,branch,tag,钩子,脚本,分支,标签,游戏开发，备忘录
---

###干货备忘录
######本文的适应环境及声明
* 发布文件夹所在的服务器上有git版本库
* 否则，只有钩子脚本可供参考
* 我不是git专家，我对git的使用观念是够用就好，本文是满足我需求的实践总结，大家随意交流，如有转载是我的荣幸！。

本文模拟一个场景：我们需要在服务器 **SERVER** 上将 */home/git/repositories/sg/xxx.git* 提交自动发布/更新到*/home/game/*下同名的文件夹下，例如的 **branchA** 分支自动发布/更新到 */home/game/branchA* ，**branchB** 分支自动发布/更新到 */home/game/branchB*。

\**/home/game*下没有相应的分支文件夹的就不自动发布。  
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
	'''取得执行git命令的输出字符'''
    args = ['git'] + args
    git = subprocess.Popen(args, stdout = subprocess.PIPE)
    details = git.stdout.read()
    details = details.strip()
    return details

def get_repo_name():
	'''读取当前提交的版本库的名称'''
    if git(['rev-parse','--is-bare-repository']) == 'true':
        name = os.path.basename(os.getcwd())
        if name.endswith('.git'):
            name = name[:-4]
        return name
    else:
        return os.path.basename(os.path.dirname(os.getcwd()))

REPO_NAME = get_repo_name()

if __name__ == '__main__':
	#读取git调用钩子时的参数
    for line in sys.stdin.xreadlines():
        old, new, ref = line.strip().split(' ')

	#取得分支名
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

		#用umask 设置git pull执行新建的文件的权限 
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

######惨痛教训
1. 在实现的过程中，可能是作者的环境权限很宽松的原因，在权限上走了些弯路，特别是git钩子pull的文件都是数组为git：git，导致其他用户没有权限读取、执行发布文件
1. 建立发布文件夹的方式也是遇到了多次挫折，网上的多篇文档都没有强调这个，看了很多个文档，经过n次尝试，最后用了本地路径协议避免了权限问题（**git clone /home/git/repositories/sg/xxx.git -b branchA branchA**）。
1. 在脚本里执行pull的那句command 也是我的”心血“：**os.system('cd '+ pullpath +'; **env -i **git log -1 --pretty=format:"%s"')**，刚开始没有加上**env -i**时，总是提示当前路径没有.git版本库。而我的python项目里经常不**env -i**也正常。我现在还不清楚原因，希望朋友们可以在回复里告诉我。

###后记
1. IT行业分工越来越细了，我从事开发十余年了（人生能有几个十年？），但现在越来越觉得自己所知甚少，很多时候都在是找个XXX岗位的专才还是自己学习学将就将就。之前一直用svn，最近换成git管理开发。享受git便利的同时，也必须接受新的挑战和折磨，~_~！
1. 这两天对的git自动发布总算有了些精力，也终于写了这篇短文，希望可以帮助到同行。

