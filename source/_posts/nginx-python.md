---
title: 基于Nginx搭建Python WSGI应用
tags:
  - python
  - nginx
  - wsgi
  - uwsgi
  - linux
date: 2016-05-25 13:27:11
categories: Nginx
---

相比于Apache，Nginx是一个轻量级高性能的服务器和反向代理，其特点是资源占用少，并发能力高。下面我就介绍一下如何在Linux下基于Nginx搭建Python wsgi应用。

<!-- more -->

# 相关知识
要完成WSGI应用的部署，需要对相关的知识有一定的了解。在这里我们需要了解的除了Nginx以外还有WSGI，uWSGI。总的来说，这三者的关系可以参考下图。如果你已经足够了解，则可以直接跳过这部分。

![](/uploads/py-nginx.png)

**Nginx 服务器**
Nginx是一个反向代理服务器(Reverse Proxy Server)。反向代理是指以代理服务器来接受Internet上的连接请求，然后将请求转发给内部网络上的服务器；并将从服务器上得到的结果返回给Internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器，同时因为这个代理服务器的作用正好和正向代理相反，因此就有了反向代理的名字。更多关于反向代理的信息，可以参考：[反向代理服务器的工作原理][reverse-proxy]。

**uWSGI 服务器**
uWSGI也是一个服务器，它和php-cgi类似，提供的就是执行对应脚本的功能，php-cgi提供php脚本的功能，uWSGI提供执行python脚本的功能，uWSGI支持多种协议，不过我们最关心的是其支持的uwsgi协议。

**WSGI 应用**
PythonWeb服务器网关接口（Python Web Server Gateway Interface)，缩写是WSGI。WSGI是Python应用程序或框架和Web服务器之间的一种接口。我们把运行在uWSGI服务上的Python程序称为WSGI应用。

# 环境准备
在继续安装之前，你需要先确保自己的服务器上安装有`Python`以及`pip`，其对应的安装方法在网上可以容易的找到。我使用的操作系统是`CentOS`使用`yum`管理和安装软件。其他平台的安装方法也大同小异，如果操作系统是`Debian`系列的，例如`Ubuntu`，则理论上只要将下面命令的`yum`改变为`apt-get`即可。

**下载并安装uWSGI**
使用`pip`安装uWSGI

```bash
$ pip install uwsgi
```

**下载并安装Nginx**
在不同Linux平台下安装的方法各不相同，我这里使用的是`CentOS`，使用的命令如下

```bash
$ sudo yum install nginx
```

# 创建WSGI应用
我们现在要创建运行于uWSGI服务器的WSGI应用。我们先使用测试用的WSGI Application。将下列内容保存在名字为`wsgi.py`的文件里。

```python
def application(env, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return ["Hello!"]
```

>  文件名`wsgi.py`不是必须的，理论上你可以使用任意文件名。

这是一个最简单的WSGI程序样例，通过观察代码我们可以知道。应用返回`Hello`字符串以及状态码`200`。关于WSGI的相关接口定义可以参照[WSGI Tutorial][wsgi-tutorial]。

我们的WSGI应用需要借助uWSGI服务器运行，使用如下命令运行在uWSGI服务器上运行我们的`wsgi.py`程序。

```bash
$ uwsgi --socket 127.0.0.1:9090 --protocol=http -w wsgi
```

这条命令会在前台直接运行uWSGI服务器，我们通过`-w wsgi`指定使用Python脚本`wsgi.py`处理HTTP请求。

**注意**：当uWSGI和Nginx一起运行时，需要移除选项`--protocol=http`，否则Nginx和uWSGI之间无法进行通信。原因是Nginx和uWSGI之间的通信并不依赖HTTP协议，而是uwsgi协议。

> 对于在前台运行的uWSGI服务器，你可以使用`Ctrl+C`来停止其运行

选项`--socket 127.0.0.1:9090`代表uWSGI监听`9090`端口，但是只接收IP地址为127.0.0.1(也就是本地IP)的请求，这是处于安全考虑的，uWSGI并不直接对外部提供服务，由我们的反向代理服务器Nginx负责代理客户端的HTTP请求。

运行这条命令以后，通过浏览器访问`http://loaclhost:9090`应该能看到对应的页面。

> 如果你不是在本地进行配置，则需要将`--socket`选项的内容指定为`0.0.0.0:9090`，表示接受来自所有ip的请求。然后在浏览器访问`http://your_ip:9090`。

uWSGI服务器运行以后，如果能够在浏览器里看到对应的页面，则代表我们的WSGI应用以及uWSGi服务器没有问题，但是目前和Nginx还没有任何关系，接下来我们需要进行Nginx相关的配置。同时，目前uWSGI的运行方式只是暂时的，要真正让uWSGI和Nginx一起稳定的工作，我们还需要对uWSGI进行更多的配置，我们会在后面的章节继续讨论。

# 配置Nginx服务器
如果uWSGI服务器已经正常运行，我们还需要配置Nginx服务器，以便两个服务器之间可以正常通信。

打开Nginx配置文件

```bash
$ sudo vim /etc/nginx/nginx.conf
```

> `/etc/nginx/nginx.conf`是大多数Nginx配置文件的默认路径，你也可以使用`sudo nginx -t`或者`sudo nginx -V`来查看自己机器上配置文件的配置。

下面给出一个完成的`nginx.conf`配置文件参考，你也可以将这个文件直接替换`nginx.conf`，但是推荐只将将自己需要的部分填入自己对应的配置文件的位置。关键的部分我会以注释给出。

```nginx
worker_processes 1;

events {

    worker_connections 1024;

}

http {

    sendfile on;

    gzip              on;
    gzip_http_version 1.0;
    gzip_proxied      any;
    gzip_min_length   500;
    gzip_disable      "MSIE [1-6]\.";
    gzip_types        text/plain text/xml text/css
                      text/comma-separated-values
                      text/javascript
                      application/x-javascript
                      application/atom+xml;

    # 之前配置的uWSGI服务器的相关信息
    # 这里Nginx允许配置多个uWSGI服务器，用于负载均衡
    upstream uwsgicluster {
        server 127.0.0.1:9090;
        # server 127.0.0.1:9091;
        # ..
        # .
    }

# Nginx的服务器配置
    server {

        # 端口
        listen 80;

        # 设置静态文件目录
        # 对于静态文件，Nginx直接将文件内容返回 
        location ^~ /static/  {

            # Example:
            # root /full/path/to/application/static/file/dir;
            root /app/static/;

        }
        
        # 作为反向代理连接到WSGI应用
        # 必需
        location / {
        
            include            uwsgi_params;    # 必需
            
            uwsgi_pass         uwsgicluster;    # 这里的uwsgicluster来自刚才指定的upstream配置，这样可以提供负载均衡的功能
            # uwsgi_pass    127.0.0.1:9090;     # 你也可以这里的代码直接指定地址

            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;

        }
    }
}
```

使用上面的配置，Nginx会将所有以`/static/`开头的访问转到对应的静态文件，将其他所有的请求转发到我们的WSGI应用。

到这里就基本上完成了Nginx的配置，输入下面的命令可以让Nginx重新载入配置。

```bash
$ sudo nginx -s reload
```

如果上面的命令在你的机器上无效，可以试一试下面的命令

```bash
$ sudo service nginx stop
$ sudo service nginx start
```

如果你还不太理解Nginx的配置文件，欢迎参考我的另一篇博客：[深入理解 Nginx 配置](/2016/05/29/nginx-config-struct/)

# 配置uWSGI服务器

之前我们已经介绍了一些运行uWSGI服务器的方法了，但是每次启动服务都需要手动将参数输入命令行是十分麻烦的，而且不方便我们调试。因此我们需要将uWSGI的配置写入文件中。

uWSGI可以支持`.ini`和`.json`的配置文件，这两个文件的配制方法大同小异。这里我使用的是`.ini`，下面直接给出我的配置文件代码。更多的配置选项说明，请参考[uWSGI配置文档翻译][uwsgi-option]

```ini
[uwsgi]

# socket = addr:port
socket = 127.0.0.1:9090

# WSGI Application 所在的目录
chdir  = /home/winton/www/py-nginx

# 是否使用master进程对worker进行管理
master = true

# woker的数量
processes = 1

# WSGI应用的文件
wsgi-file = wsgi.py

# 使进程在后台运行，并将日志打到指定的日志文件
daemonize = /tmp/uwsgi/uwsgi_deamonize.log

# 将pid写到指定的pidfile文件中
pidfile = /tmp/uwsgi/uwsgi_pid.pid

```

配置文件写好之后，我们可以直接使用如下命令运行我们的uWSGI服务器。

```bash
$ uwsgi --py-autoreload=1 --ini wsgi.ini
```

这个命令会直接在后台运行我们的WSGI应用，其中`--py-autoreload=1`会检测`.py`文件的改动并重新载入对应的文件。在开发时是很有用的一个选项。


现在，我们已经完成所有搭建工作，接下来就是继续完善我们的WSGI应用了。你可以参考[WSGI Tutorial][wsgi-tutorial]来了解如何调用WSGI接口来处理`GET`和`POST`请求。也可参考我的[github仓库][github-py-nginx]，这里有涉及到的所有代码。

# 参考资料
> [How to Deploy Python WSGI Applications Using uWSGI Web Server with Nginx] [nginx-python]
> [Quickstart for Python/WSGI applications][uwsgi-doc]
> [WSGI Tutorial][wsgi-tutorial]
> [uWSGI配置文档翻译][uwsgi-option]

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。

[nginx-python]: https://www.digitalocean.com/community/tutorials/how-to-deploy-python-wsgi-applications-using-uwsgi-web-server-with-nginx

[uwsgi-doc]: http://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html

[wsgi-tutorial]: http://wsgi.tutorial.codepoint.net/intro

[uwsgi-option]: http://www.cnblogs.com/zhouej/archive/2012/03/25/2379646.html

[reverse-proxy]: http://blog.csdn.net/keyeagle/article/details/6723408/

[github-py-nginx]: https://github.com/Heaven1881/py-nginx
