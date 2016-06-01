---
title: 深入理解 Nginx 配置
tags:
  - nginx
  - 配置
categories: Nginx
date: 2016-05-29 13:23:22
---


在本文，我们将讨论Nginx的基础配置结构，来让大家更好的理解Nginx的逻辑。

<!-- more -->

> 原文链接 [Understanding the Nginx][understanding-nginx]
> 本文主要结合这篇文章的内容和我自己的一些理解

Nginx是一个高性能的Web服务器。除了可以很好的处理Web静态内容，Nginx在高并发的环境下表现也十分突出。Nginx是世界上最受青睐的服务器之一，它不仅是普通的Web服务器，还可以是邮件服务器以及反向代理服务器。

在本文，我们将讨论Nginx的基础配置结构，来让大家更好的理解Nginx的逻辑。

Nginx在逻辑上将组织为一个个包含嵌套和并列关系的结构，在这里我们把这样一个个的结构统称为域(Context)。对于Nginx来说每次处理客户端的请求就等价于在配置中选择能够匹配请求的域，而Nginx进行选择的过程就是我们在这篇文章需要讨论的内容。

# 主要域(Context)
如果你查看 Nginx 的配置文件(例如:`/etx/nginx/nginx.conf`)，你会发现每个域都是由`{`和`}`包裹起来的数据集，它们以类似于树的结构进行组织。域的内部则是由多条指令组成。一个域可以被另一个域包含，在这种情况下，上一级域指令会被传递到下一级域。指令是对Nginx进行配置的主要方法。在本文，我们会涉及到一些常用的指令，你也可以查阅[Alphabetical index of directives][nginx-directive-doc]了解更详细的信息。

## Main 域
Main域也就是我们的全局环境，所有域的最外面就是Main域，在Nginx配置中，Main域的位置就像下面这样

```nginx
# 这里就是main域，处于所有域的外面

...
context {
    ...
}
```

任何出现在main域的指令都被称作“全局指令”。需要注意的是如果你的Nginx是根据不同模块进行配置的，例如一般我们会将每个server的配置分开写到`conf.d/*.conf`或者`server/*.conf`，在这些文件里，可能会包含一些看起来处于所有域外的指令，但是实际上这些指令并不是全局指令，它们会在其它文件中被引入到另一个域中。

main域可以用来存放一些设计到全局的，基础的指令。全局的指令的值会为子域提供默认值，子域也可以根据需要对相应的指令进行重载。

## Event 域
Event域是Main域的一个子域，用于配置Nginx处理连接的方式，在Nginx配置文件中只能有一个Event域。Nginx使用基于事件的连接处理模型，一般在Event域中的指令都是用来选择worker处理连接的相关方法，或者用来修改对应方法的实现。

Event域还可以用来配置其他的一些信息，例如worker同时处理的连接数，以及是否让每个worker一次只处理一个连接，是否让worker轮流处理连接。

## Http 域
如果我们希望让Nginx作为一个Web服务器或者反向代理服务器，Http域就会被用来保存各种相关的配置，Http域里保存所有用来处理HTTP/HTTPS请求的指令。

和Event域一样，Http域必须直接是Main域的子域。

大部分更具体的配置被保存在Http域下面的Server域，在Http域里主要设置每个Server域的默认值，例如：相关的日志文件的保存位置(`access_log`，`error_log`)，文件的异步I/O操作(`aio`, `sendfile`, `directio`)，服务器不同状态码对应的页面(`error_page`)，压缩选项(`gzip`和`gzip_disable`)。

## Server 域
Server域处于Http域内部，Nginx允许同时定义多个Server域，他们在Nginx配置文件中的大概就像下面的格式。

```nginx
# main context
http: {
    # http context
    server {
        # first server context
    }
    server {
        # second server context
    }
}

```

每个Server域代表了一个可以处理客户端请求的虚拟服务器。因为Nginx可以同时指定多个虚拟服务器，因此Nginx需要根据用户的请求来决定让哪一个Server进行处理。

Nginx通过下面两个字段来判断一个Server是否应该被用来处理某一个客户端请求。

- `listen`：IP和端口，Nginx会优先选择这一字段匹配的Server
- `server_name`：域名。如果多个Server有相同的`listen`值，Nginx则进而匹配这一字段。

**Nginx 如何选择Server来处理请求**
首先，Nginx检查客户端的IP地址和访问的端口号，从Server列表中根据`listen`字段筛选出匹配的Server。

`listen`记录了每个Server选择监听的ip地址和端口，如果一个Server没有指定`listen`，则会取默认值`0.0.0.0:80`(如果Nginx不是以`root`身份运行，则默认值为`0.0.0.0:8080`)

`listen`字段可以设置为如下格式

- IP和端口的组合
- 只有IP，这种情况下端口会取默认值`80`
- 只有端口，这种情况下IP取默认值`0.0.0.0`
- UNIX socket的路径

Nginx检查`listen`字段的步骤和规则如下

1. 将“不完整”的`listen`转换为完整的`listen`值
 - 如果一个Server没有指定`listen`，则`listen`的值取`0.0.0.0:80`
 - `111.111.111.111`将被转换为`111.111.111.111：80`
 - `8000`将转换为`0.0.0.0:8000`
2. 从所有备选Server中根据`listen`字段选出最匹配的Server，如果有多个最匹配的Server，则将其都选入待选列表中。需要注意的是，我们这里说的是“最匹配”，也就是说，在端口都相同的情况下，如果有Server监听的IP刚好等于客户端的IP，那么其余所有监听`0.0.0.0`的Server都不会被加入待选列表。
3. 经过上面的筛选，如果只剩余一个Server，则Nginx会直接忽略`server_name`选择这一个Server处理客户端请求，如果有多个Server，则Nginx会继续比较`server_name`字段

在下面的例子中，假设客户端的IP为`192.168。1.10`，服务器的域名为`example.com`，客户端访问域名`example.com`，则最后只有第一个服务器被选择来处理请求，因为`listen`字段一旦匹配则不再考虑`server_name`字段。

```nginx
server {
    listen 192.168.1.10;
    . . .
}

server {
    listen 80;
    server_name example.com
    . . .
}
```

如果有多个Server的`listen`字段一样，那么Nginx则会开始比较`server_name`字段。

> Nginx通过检查客户端发送的HTTP请求的“Host”头部来确定客户端访问的域名，这个字段会用来和`server_name`比较。

Nginx根据以下顺序来选择匹配的Server：

- 寻找精确匹配的`server_name`，如果存在，则选择第一个匹配的Server
- 寻找使用前置通配符`*`的`server_name`，如果能够匹配多个Server，则选择最长匹配
- 寻找使用后置通配符`*`的`server_name`，如果能够匹配多个Server，则选择最长匹配
- 寻找使用正则表达式匹配的`server_name`，如果有多个Server匹配，选择第一个匹配的Server
- 如果经过上面的流程，没有Server复合要求，则Nginx会选择默认的Server

> 如果需要在`server_name`中使用正则表达式，则需要在表达式之前使用`~`声明。

Nginx通过`default_server`来指定默认的Server，否则使用第一个Server作为默认Server。需要注意的是，每一个IP和端口的组合只能指定一个`default_server`，只有`listen`字段符合要求才有可能会选为默认Server。

在下面的例子中，如果请求的域名是“cyberpunk.example.com”，则第二个Server会被选中处理请求。

```nginx
server {
    listen 80;
    server_name *.example.com;
    . . .
}

server {
    listen 80;
    server_name cyberpunk.example.com;
    . . .
}
```

如果Nginx没有找到精确匹配的`server_name`，则会寻找使用前置通配符的`server_name`，如果有多个匹配，则选择最长匹配的Server。在下面的例子中，如果访问的域名是`www.example.com`，则第二个Server会被选中。

```nginx
server {
    listen 80;
    server_name www.example.*;
    . . .
}

server {
    listen 80;
    server_name *.example.org;
    . . .
}

server {
    listen 80;
    server_name *.org;
    . . .
}
```

如果没有找到匹配的`server_name`，Nginx会检查后置通配符的匹配情况，同样也会选择最长匹配。下面的例子中，访问的域名为`www.example.com`，第三个Server会被选择。

```nginx
server {
    listen 80;
    server_name host1.example.com;
    . . .
}

server {
    listen 80;
    server_name example.com;
    . . .
}

server {
    listen 80;
    server_name www.example.*;
    . . .
}

```

如果没有找到匹配的后置通配符`server_name`，下一步Nginx会寻找正则表达式。Nginx会选择第一个匹配的Server。在下面的例子中，客户端访问的域名为`www.example.com`，第二个Server会被选中。

```nginx
server {
    listen 80;
    server_name example.com;
    . . .
}

server {
    listen 80;
    server_name ~^(www|host1).*\.example\.com$;
    . . .
}

server {
    listen 80;
    server_name ~^(subdomain|set|www|host1).*\.example\.com$;
    . . .
}
```

如果上面的步骤无法选择出对应的Server，那么客户端的请求将会交由匹配`listen`字段的默认Server处理。

## Location 域
一旦Nginx完成Server域的选择，接下来的工作就是选择对应Location域。每个Server可以定义多个Location域，每个Location对应一种类型的客户端请求，每个Location的格式如下

```nginx
location (可选修饰符) (匹配字符串) {
    . . . 
}
```
匹配字符串用于和客户端请求的URI作比较，可选修饰符会影响Nginx匹配URI的方式，可选修饰符的类型如下：

- 无可选修饰符：在这种情况下，匹配字符串会被用于前缀匹配，即从URI的头部开始匹配
- `=`：精确匹配
- `~`：大小写敏感的正则表达式匹配
- `~*`：大小写不敏感的正则表达式匹配
- `^~`：如果匹配字符串是最佳的非正则表达式匹配，则不再进行其他正则表达式的匹配判断

> URI的例子：对于访问`http://www.example.com/blog`，其URI为`/blog`

Location域处于Server域之下，与Server域不同的是，Location域允许相互嵌套，因此，我们可以借助这一特性去逐步拆解客户端访问的URI，提供更为清晰的管理方式。

```nginx
# main context
server {
    # server context
    location /match/criteria {
        # first location context
    }
    
    location /other/criteria {
        # second location context
        location nested_match {
            # first nested location
        }

        location other_nested {
            # second nested location
        }
    }
}
```

**Location的例子**
Location默认的匹配是前缀匹配，下面的Location可以匹配`/site`，`/site/page/index.html`和`/site/index.html`

```nginx
location /site {
    . . .
}
```

使用`=`，可以修改Location的匹配方式为精确匹配，下面的例子只能匹配`/page`，不能匹配`/page/index.html`。不过这里需要注意的是，如果我们使用`index index.html`的指令来指定默认页面，则进入这个Loaction后，Nginx会先将URI补充为`/page/index.html`，然后会通过一个重定向跳转到另外一个可以处理这个URI的Location。

```nginx
location = /page {
    . . .
}
```

下面的Location可以匹配`a.png`，但是不能匹配`a.PNG`

```nginx
location ~ \.(jpe?g|png|gif|ico)$ {
    . . .
}
```

下面的Location可以匹配`a.png`和`a.PNG`

```nginx
location ~* \.(jpe?g|png|gif|ico)$ {
    . . .
}
```

下面的Location如果被选为最佳的非正则表达式匹配，则其会阻止正则表达匹配的进行。它也可以匹配请求`/customes/ninjia.html`

```nginx
location ^~ /costumes {
    . . .
}
```

**Nginx选择Location的顺序**
1. 精确匹配，使用`=`修饰的Loaction，如果其恰好匹配客户端请求的URI，则这个Location会被直接选择并结束搜索。
2. 如果没有精确匹配，则Nginx会寻找最长的前缀匹配。如果寻找到的匹配包含修饰符`^~`，则直接结束搜索并选择这个Location作为最终选择，否则，将这个Location加入备选列表中，在遍历所有可能的Loaction后，从列表中选择最长的前缀匹配。
3. 选择出最长的前缀匹配后，Nginx会继续进行正则表达式的Location匹配(包括大小写敏感和大小写不敏感的正则表达式匹配)。在这一步中，Nginx会选择第一个匹配的正则表达式的Location。
4. 如果在步骤3中找到了匹配的正则表达式，则使用其对应的Location来处理请求。如果没有找到匹配的正则表达式，则使用步骤2中的最长前缀匹配Location。

和`server_name`稍有不同的是，默认情况下，正则表达式匹配的优先级比前缀匹配更高，你可以通过修饰符`^~`来提高前缀匹配的优先级。

**Location之间的跳转条件**
在多数情况，一个Location选择以后，接下来的操作都会以来于Location内部的指令来工作，与Location外部几乎没有关系。不过还是有少数指令可以实现Location之间的跳转，我们称之为内部重定向(internal redirect)。

- index
- try_files
- rewrite
- error_page

`index`会始终触发一个内部重定向，特别是在我们使用修饰符为`=`的精确匹配时。一般情况下使用精确匹配可以加快Nginx匹配请求的速度，因为精确匹配一旦达成，则会直接终止搜索。但是，如果我们精确匹配的路径是一个目录，那么这个请求很可能会被内部重定向到其他的Location。

在下面的例子中，客户端访问的URI为`/exact`，虽然这个请求和第一个Locatoin精确匹配，但是由于`index`的作用，请求最终会跳转到第二个Location处理。

```nginx
index index.html;

location = /exact {
    . . .
}

location / {
    . . .
}
```

在这个种情况下，如果你真的需要让URI为`/exact`的请求匹配到第一个Location，可以通过设置一个非法的`index`值，同时将`autoindex`设置为`on`。

```nginx
location = /exact {
    index nothing_will_match;
    autoindex on;
}

location  / {
    . . .
}
```

这是组织`index`触发内部重定向的一种方法，不过在大部分Nginx配置文件中并不常见。

`try_files`会依次检查文件和目录的是否存在，其最后一个参数可以是URI的形式，如果前面列举的文件或者目录都不存在，那么Nginx会触发一个内部重定向到这个URI。

```nginx
root /var/www/main;
location / {
    try_files $uri $uri.html $uri/ /fallback/index.html;
}

location /fallback {
    root /var/www/another;
}
```

在上面的例子中，假设请求的URI为`/balabala`，那么首先第一个Location会用于处理这个请求，Nginx会在目录`/var/www/main/`下依次检查`balabala`，`balabala.html`，`balabala/`是否存在，如果上面的检查都失败了，URI会变为`/fallback/index.html`，这会重新触发Location的选择。最终这个请求会交由第二个Location处理，Nginx会返回文件`/var/www/another/fallback/index.html`。

参考上面的例子，我们在第一个Location加入了`rewrite`指令，我们可以发现，有时候请求会在执行`try_files`之前被直接发送到第二个Location。

```nginx
root /var/www/main;
location / {
    rewrite ^/rewriteme/(.*)$ /$1 last;
    try_files $uri $uri.html $uri/ /fallback/index.html;
}

location /fallback {
    root /var/www/another;
}
```

上面的例子中，如果访问的URI是`/rewriteme/hello`，经过`rewrite`指令后URI变为`/hello`，仍然在第一个Location内，因此`try_files`会继续执行，如果三个文件都检查失败(就想我们之前讨论的那样)，那么请求会跳转到第二个Location。

如果访问的URI是`/rewriteme/fallback/hello`，经过`rewrite`以后会变为`/fallback/hello`，因此这个请求会直接发送给第二个Location，跳过`try_files`。

和`rewrite`功能类似的还有`return`，`return`通过指定状态码为`301`或者`307`和对应的URL，也可以实现重定向，不过的`return`会使客户端重新发送请求。

`error_page`用于指定返回的状态码对应的URI。当Location中还有`try_files`时，`error_page`就很可能不会被执行，因为`try_files`已经处理了整个的请求过程。

我们来看下面一段配置。

```nginx
root /var/www/main;

location / {
    error_page 404 /another/whoops.html;
}

location /another {
    root /var/www;
}
```

除了开头为`/another`的请求，所有的请求都会被第一个Location处理，如果客户端请求了一个不存在的文件，Nginx就会返回一个`404`的状态码，在这个情况下，URI会被内部重定向到`/another/whoops.html`，交由第二个Location处理。

# 其他域
除了上面介绍的，Nginx还有很多域，例如：

- `upstream`：Nginx作为反向代理服务器，可以使用这个域作为负载均衡的控制配置。这也是一个很重要的Context，以后我会详细介绍。
- `mail`：用于配置邮件服务器。
- `if`：一般出现在`Location`域中，用于条件控制，大多数情况都可以用更直观的`rewrite`代替`if`，因此建议尽量少用`if`。
- `limit_except`：用于增加对访问权限的控制。



[understanding-nginx]: https://n0where.net/understanding-the-nginx/

[nginx-directive-doc]: http://nginx.org/en/docs/dirindex.html
