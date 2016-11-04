---
title: 给Hexo的Next主题增加统计访客次数的功能
tags:
  - hexo
  - next
  - 访客统计
  - leancloud
categories: Hexo
date: 2016-05-31 16:09:13
---


之前参考[这篇博客][教程]给自己的博客增加了访客统计功能，不过最新版本的`Next`主题已经可以支持访客统计功能。

<!-- more -->

最终的效果如下，现在来介绍具体的步骤。
![][最终效果图片]

**注意**：本教程只适用于Hexo博客的Next主题，如果不是Next主题，参考[Hexo的NexT主题个性化：添加文章阅读量][教程]，这篇文章提供了一个适用于所有Hexo博客的方法。使用本教程时，最好将Next主题更新到最新版本。我使用的版本为`5.0.1`。可以查看`/theme/next/_config.yml`文件来确认自己的Next主题版本。

# 配置LeanCloud

打开[LeanCloud官网][leancloud]，注册并完成邮箱激活后，进入控制台，创建一个新应用

![][创建新应用]

进入我们新建的应用，进入`存储`分页，创建名为`Counter`的Class。

![][创建Class]

![][创建Class-2]

> 在这里，我们创建的Class名字必须为`Counter`，因为NexT会固定访问名字为`Counter`的Class，这个名字应该可以通过修改配置来实现更改，但是目前我找不到修改的方法。

# 配置Hexo
打开`/theme/next/_config.yml`，在如下对应位置填入`app_id`和`app_key`，并将`enable`设置为`true`。

```yaml
# Show number of visitors to each article.
# You can visit https://leancloud.cn get AppID and AppKey.
leancloud_visitors:
  enable: true
  app_id: # your app_id
  app_key: # your app_key
```

AppID和AppKey可以通过LeanCloud应用的`设置`分页下的`应用key`页面找到。

![][应用key的位置]

# 设置安全域名
有时候我们的会在本地通过`locahost:4000`浏览并编辑我们的页面，在这种情况下，LeanClound会记录很多没有意义的浏览次数。为了让统计的浏览次数有意义，我们可以在`应用->设置->安全中心->Web安全域名`中设置自己博客的域名，只有该域名可以访问LeanCloud系统，因此只会记录在这个域名下的访客数据。

![][设置安全域名]

完成上面所有的设置后，重新部署你的博客，应该可以看到对应的效果。

# 在底部显示总访问量标签
这一部分的效果如下，大家可以根据自己的需求决定取舍。

![][底部标签]

打开`/theme/next/layout/_partials/footer.swig`，在底部添加如下代码

```html
<script async src="https://dn-lbstatics.qbox.me/busuanzi/2.3/busuanzi.pure.mini.js"></script>
<span id="busuanzi_container_site_pv">
    &nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;总访问量: <span id="busuanzi_value_site_pv"></span>
</span>
```
> 引入的JS脚本来自@不蒜子，这个脚本通过上传访问网站时的域名来实现记录访问量的功能，在平时测试时，如果我们使用类似于`localhost`或者`127.0.0.1`的地址，会发现访问量变得很大，如果使用自己的域名则不会有这个问题。

# 参考资料
> [Hexo的NexT主题个性化：添加文章阅读量][教程]


[教程]: http://www.jeyzhang.com/hexo-next-add-post-views.html
[leancloud]: https://leancloud.cn

[最终效果图片]: /uploads/hexo-counter/final.png
[创建新应用]: /uploads/hexo-counter/new-app.png
[创建Class]: /uploads/hexo-counter/new-class.png
[创建Class-2]: /uploads/hexo-counter/new-class-2.png
[应用key的位置]: /uploads/hexo-counter/app-key-position.png
[设置安全域名]: /uploads/hexo-counter/set-host.png
[底部标签]: /uploads/hexo-counter/footer.png