---
layout: post
title: 用Expect实现多台机器SSH无密码登陆
category: server
tags: Linux@server
keywords: expect ssh login
description: 
from: 

---

公司新安装了十几台服务器，为了方便服务端安装程序、拷贝数据，几台机器互相SSH、SCP都要输入密码的话那就太麻烦了，所以要用[SSH RSA密钥][SSH_RSA]进行自动登陆验证。

{% highlight sh %}
#首先需要生成RSA密钥
[root@localhost ~]# ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
  same passphrase again: 
  identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
XX:XX:YY:YY:ZZ:ZZ:XX:YY:ZZ:ZZ:YY:YY:XX:AA:66:FF root@localhost
{% endhighlight %}

{% highlight sh %}
#然后把密钥传输到其他服务器
[root@localhost ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub  root@nginx.lkfm.com
21
The authenticity of host '1.lialun.com (192.168.3.10)' can't be established.
RSA key fingerprint is XX:XX:YY:YY:ZZ:ZZ:XX:YY:ZZ:ZZ:YY:YY:XX:AA:66:FF
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '1.lialun.com (192.168.3.10)' (RSA) to the list of known hosts.
root@1.lialun.com: 
Now try logging into the machine, with "ssh '1.lialun.com'", and check in:
  .ssh/authorized_keys
to make sure we haven't added extra keys that you weren't expecting.
{% endhighlight %}

一两台机器还好，如果几十台甚至更多机器，那这个传输密钥的工作量实在是太大了，于是考虑写脚本解决这个问题。
由于需要用户输入指令，找了半天，没有什么简单的办法，只有[EXPECT][expect]可以实现这个功能。于是写脚本如下：

{% highlight sh %}
#!/usr/bin/expect

set passwd 123456

# remove old ssh key
spawn rm -f /root/.ssh/id_rsa /root/.ssh/id_rsa.pub /root/.ssh/id_rsa.pub /root/.ssh/known_hosts
expect {
}

# generate ssh key
spawn ssh-keygen -t rsa
expect { 
	"*id_rsa*" { send "\r"; exp_continue } 
	"*passphrase*" { send "\r"; exp_continue } 
	"*again*" { send "\r" } 
} 

# send to others
foreach domain {
	1.lialun.com
	2.lialun.com
	3.lialun.com
} {
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub  $domain
expect {
	"*connecting*" { send "yes\r"; exp_continue }
	"*asswor*" { send "$passwd\r" } 
}
}
{% endhighlight %}

运行后发现密码输入不进去，尝试了好久找不到思路，后来看到这篇文章：

[spawn-expect-send-and-interact][spawn-expect-send-and-interact]

[can-the-expect-script-continue-to-execute-other-command-after-interact][can-the-expect-script-continue-to-execute-other-command-after-interact]

于是尝试在输入完成密码后进行interact，代码如下：

{% highlight sh %}
#!/usr/bin/expect

set passwd 123456

# remove old ssh key
spawn rm -f /root/.ssh/id_rsa /root/.ssh/id_rsa.pub /root/.ssh/id_rsa.pub /root/.ssh/known_hosts
expect {
}

# generate ssh key
spawn ssh-keygen -t rsa
expect { 
	"*id_rsa*" { send "\r"; exp_continue } 
	"*passphrase*" { send "\r"; exp_continue } 
	"*again*" { send "\r" } 
} 

# send to others
foreach domain {
	1.lialun.com
	2.lialun.com
	3.lialun.com
} {
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub  $domain
expect {
	"*connecting*" { send "yes\r"; exp_continue }
	"*asswor*" { send "$passwd\r" } 
}
interact
}
{% endhighlight %}

测试成功。果然是因为没有interact导致的。

[SSH_RSA]:http://blog.csdn.net/wangjunjun2008/article/details/20037101
[expect]:http://blog.csdn.net/leexide/article/details/17485451

[spawn-expect-send-and-interact]:http://avdeo.com/2009/08/14/spawn-expect-send-and-interact/
[can-the-expect-script-continue-to-execute-other-command-after-interact]:http://stackoverflow.com/questions/7568738/can-the-expect-script-continue-to-execute-other-command-after-interact