---
layout: post
title: Mac OSX 10.8.1 安装 Python2.7(Django)
---

Mac OSX 10.8.1 默认的 Python 版本是 2.6.8，虽然 3.2 出来已经有较长时间了，但现主流的版本仍然是 2.X 系例版本，我这里升级为 2.7.3 版本，其对应的 Django 版本为 1.4.1。 

##### 环境准备

* 安装 Xcode 最新版本(4.4)
* 安装命令行工具: [Command Line Tools](http://cxwangyi.wordpress.com/2012/03/26/xcode-4-3-command-line-tools/)
* 安装 [Homebrew](http://mxcl.github.com/homebrew/)

##### Python安装

以上环境安装好之后，即可通过 brew 安装 pyhton：
{% highlight bash %}
$ brew install readline sqlite gdbm
$ brew install python --universal --framework
{% endhighlight bash %}

安装完成后，需要设置链接(Symlinks)：
{% highlight bash %}
$ mkdir ~/Frameworks
$ ln -s "/usr/local/Cellar/python/2.7.3/Frameworks/Python.framework" ~/Frameworks
$ ln -s /usr/local/Cellar/python/2.7.3/bin/python2.7 /usr/local/bin/python2.7
{% endhighlight bash %}

安装 [pip](http://pypi.python.org/pypi/pip/) 和 [Virtualenv](http://www.openfoundry.org/tw/tech-column/8516-pythons-virtual-environment-and-multi-version-programming-tools-virtualenv-and-pythonbrew)  & Django：
{% highlight bash %}
$ /usr/local/share/python/easy_install pip
$ /usr/local/share/python/pip install --upgrade distribute
$ /usr/local/share/python/pip install virtualenv
$ /usr/local/share/python/pip install virtualenvwrapper
$ /usr/local/share/python/pip install django
{% endhighlight bash %}

配置环境，编辑 ~/.bashrc，添加如下内容后执行 `source ~/.bashrc`：
{% highlight bash %}
alias python="python2.7"

# Before other PATHs...
PATH=${PATH}:/usr/local/share/python

# Python
export WORKON_HOME=$HOME/.virtualenvs
export VIRTUALENVWRAPPER_PYTHON=/usr/local/bin/python2.7
export VIRTUALENVWRAPPER_VIRTUALENV_ARGS='--no-site-packages'
export PIP_VIRTUALENV_BASE=$WORKON_HOME
export PIP_RESPECT_VIRTUALENV=true
if [[ -r /usr/local/share/python/virtualenvwrapper.sh ]]; then
    source /usr/local/share/python/virtualenvwrapper.sh
else
    echo "WARNING: Can't find virtualenvwrapper.sh"
fi
{% endhighlight bash %}
由于 Terminal 在启动时加载的用户配置并非 .bashrc，而是 ~/.bash_profile，所在还需要在 ~/.bash_profile 加入 `[ -r ~/.bashrc ] && source ~/.bashrc` 语句。

接着运行 `python --version` 查看当前版本，如果一切正常则会显示 2.7 的版本号。可安装 `ipyhton` 来验证 django 是否安装正确：
{% highlight python %}
> import django
> print django.get_version()
1.4.1
{% endhighlight python %}

环境配置好后，如果直接启动 Django 项目，会提示如下错误：
{% highlight bash %}
denger@Macbook apiworks$ python manage.py runserver
Traceback (most recent call last):
  File "manage.py", line 8, in <module>
    from django.core.management import execute_from_command_line
ImportError: No module named django.core.management
{% endhighlight bash %}
原因是 manage.py 的第一行代码中, `#!/usr/bin/env python` 使用的是默认的环境，而默认的环境下的 python 并非 2.7, 其环境下也没有 django 模块，所以提示如上错误。

解决以上问题有很多方法，最好的方法是通过 virtualenv 虚拟一个开发环境，即在任意目录下执行以下命令：
{% highlight bash %}
$ virtualenv dev-env # 可产生一个干净的 python 环境
$ cd dev-env 
$ source bin/activate # 激活环境
(dev-env)$ pip install django # 重新在该环境下安装 django
{% endhighlight bash %}
在虚拟环境下安装 django 之后，进入 project 目录下便可正常启动 django 服务 `python manage.py runserver`。当然，如果想回到系统默认环境的话只需要运行 `deactivate` 即可。
