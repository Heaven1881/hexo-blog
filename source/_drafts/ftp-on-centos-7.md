---
title: 在CentOS 7上部署FTP服务器
date: 2017-06-20 20:46:14
tags:
 - CentOS
 - FTP
categories: 其它
---

最近心血来潮想要在服务器上搭建一个FTP服务，目的是想搭建一个自己的网盘，不过在这个过程还是遇到了很多问题，好在大部分可以通过搜索引擎找到答案。下面收录和整理我在这个过程中遇到的问题：

<!-- more -->

# 安装FTP服务器

```bash
$ yum check-update
$ sudo yum install vsftpd
```

# 配置FTP服务器
FTP服务器的配置文件路径为`/etc/vsftpd/vsftpd.conf`，在修改文件前，最好先先备份原有配置

```bash
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.orig
```

大部分的FTP配置我们都不需要改，下面是需要注意的几项配置：

```bash
anonymous_enable=NO         # 禁止匿名用户登录
local_enable=YES            # 允许本地用户登录，允许/etc/passwd中指定的所有用户登录
write_enable=YES            # 允许用户修改文件系统
local_umask=022             # 文件创建掩码
dirmessage_enable=YES       # 当用户进入一个目录时，显示这个目录的信息
```

使用如下选项以限制FTP用户只能在自己的目录下操作

```
chroot_local_user=YES
allow_writeable_chroot=YES
```

> 关于FTP配置，更详细的信息可以参考：https://linux.die.net/man/5/vsftpd.conf

# 重启FTP服务器
```
$ sudo service vsftpd restart
```

# 创建FTP账号
FTP服务器使用用户账号来作为登录凭据，因此，你可以使用你的本机账号来登录FTP，或者你可以创建一个FTP专用的账号

```bash
useradd public -s /sbin/onlogin
passwd public
```

通过将用户`ftpuser`使用的shell指定为`/sbin/onlogin`，这样此用户就不能通过ssh远程连接到服务器。

> 你还可以通过修改配置`/etc/vsftpd/user_list`来指定允许登录FTP的黑名单(或者白名单)
> 关于FTP配置，更详细的信息可以参考：https://linux.die.net/man/5/vsftpd.conf

# 配置相关的安全策略
使用如下命令让FTP服务器读写本地用户的`HOME`目录

```
setsebool -P ftp_home_dir on
```
不过在我的机器上，输出：

```
setsebool:  SELinux is disabled.
```

说明本机上的SELinux处于禁用状态。

# 开启FTP端口
由于我使用的是阿里云云服务器，其默认的安全策略是禁止对端口21的访问的，因此需要进行相关配置，这一步不同的服务器会有不同的方法，不过最终的目标都是让外界可以访问本机的21端口。

# 登录FTP服务器
重启FTP服务器，尝试在外网访问FTP服务器

```bash
$ ftp 120.77.80.112
Connected to 120.77.80.112.
220 (vsFTPd 3.0.2)
Name (120.77.80.112:Heaven): public
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||15578|).
ftp: Can't connect to `120.77.80.112': Operation timed out
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
```

上面的输出出现了一次超时，注意到信息`Entering Extended Passive Mode (|||15578|).`。

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。
