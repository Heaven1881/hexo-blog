---
title: Hexo博客-NexT主题：让Hexo自动生成sitemap
tags:
  - hexo
  - sitemap
categories: Hexo博客-NexT主题
date: 2016-07-10 19:55:16
---


站点地图里面保存了当前网站希望和需要搜索引擎抓取的链接，如果我们希望自己的博客可以被Google和百度等搜索引擎搜索到，那么我们最好可以为搜索引擎Spider提供站点地图。

<!-- more -->

> *正在使用Hexo和NexT搭建你的博客吗？[点击这里](/categories/Hexo博客-NexT主题/)查看这个分类的更多文章。*

站点地图的示例可以参考http://lfwen.site/sitemap.xml ，为了保持站点地图始终是最新的，我们需要在每次发布新的文章以后及时更新对应的站点地图，但是每次都手动编辑站点地图是十分麻烦的，我们可以借助插件`hexo-generator-sitemap`来在每次部署博客时自动生成对应的站点地图。

# 安装插件

在你的Hexo博客的根目录下执行以下命令来为你的Hexo安装`hexo-generator-sitemap`插件。

```bash
$ npm install hexo-generator-sitemap --save
```

使用如下命令清除当前的缓存，并重新生成并部署博客

```bash
$ hexo clean
$ hexo g -d
```

如果一切正常，你可以通过URL`http://lfwen.site/sitemap.xml`访问到对应的站点地图(这里需要将域名换成你自己的域名)。

# 设置站点地图的名称
一般来说站点地图使用的`sitemap.xml`作为其默认名字，你也可以自己配置使用的文件名。
在博客根目录的`_config.yml`文件下添加以下文本，你可以将`sitemap.xml`指定为任意你需要的文件名。

```
sitemap:
    path: sitemap.xml
```

# 自定义站点地图
插件自动生成的`sitemap.xml`会包括网页中的大部分页面，但是并不是我们所有的网站都希望加入站点地图，例如[标签页面](http://lfwen.site/tags/)和[分类页面](http://lfwen.site/categories/)，对于类似的页面，只需要在对应的md文件的前置描述区添加`sitemap: false`即可。

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。