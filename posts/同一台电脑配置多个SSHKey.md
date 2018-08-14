##### 同一台电脑配置多个SSHKey

公司有自己的GitLab，但是我平时也喜欢给GitHub上放点东西，包括这个博客就是在GitHub托管的。因为在部署博客的时候只能使用SSH，所以就需要在公司电脑上配置两个SSHKey，那么到底我们在使用SHH的时候，它用的是哪个Key呢。在经过学习后发现SSH有一个config文件可以配置。

~~~
# GitHub的配置信息
Host github.com
HostName ssh.github.com
Port 443
PreferredAuthentications publickey
IdentityFile ~/.ssh/github_rsa  //对应为GitHub生成的KEY


# 自己公司的配置信息
Host 公司Git的host
HostName 公司Git的hostname
PreferredAuthentications publickey
IdentityFile ~/.ssh/id_rsa //对应为公司Git生成的KEY
~~~

用上面内容创建一个名为config的文件，放入~/.ssh目录里面即可。

下面是我的~/.ssh目录里面的东西

~~~
config		github_rsa.pub	id_rsa.pub
github_rsa	id_rsa		known_hosts
~~~

##### 写一个切换Git用户名邮箱信息的小脚本

有这么一个需求，就是我经常会在公司Git和GitHub切换提交东西，我希望我能快速切换Git账号信息和邮箱信息，所以就写了一个shell小脚本。

~~~Shell
#!/bin/bash
if [[ $1 == "g" ]]; then
	#statements
	git config --global user.email allenwooop@gmail.com
    git config --global user.name Woooop
	echo "设置Git邮箱为allenwooop@gmail.com"
fi
if [[ $1 == "p" ]]; then
	#statements
	git config --global user.email 私有邮箱
    git config --global user.name 私有name
	echo "设置Git邮箱为xxxx@mail.com"
fi
~~~

使用方法：自己创建一个.sh的文件，把代码粘贴进去，填写自己对应的信息。

切换为GitHub邮箱环境

~~~
./文件名.sh g //g是参数，代表GitHub
~~~

切换为私有邮箱环境

~~~
./文件名.sh p //p是参数，代表私有环境Private
~~~

