---
title: Hexo博客-NexT主题：使用leancloud进行页面访客统计
tags:
  - hexo
  - next
  - 访客统计
  - leancloud
categories: Hexo博客-NexT主题
date: 2016-05-31 16:09:13
---


之前参考[这篇博客][教程]给自己的博客增加了访客统计功能，不过最新版本的`Next`主题已经可以支持访客统计功能。

<!-- more -->

> *正在使用Hexo和NexT搭建你的博客吗？[点击这里](/categories/Hexo博客-NexT主题/)查看这个分类的更多文章。*

# 最终效果

在每篇文章的头部显示阅读次数

![][最终效果图片]

# 注意事项
在使用这篇博客之前，你需要先确保自己使用的Hexo博客的NexT主题。旧版的NexT主题可能不支持这篇文章提到的功能，在进行进一步的操作之前，确保自己使用的NexT版本支持对应功能。在这里，我使用的版本为`5.0.1`，你可以通过查看`/theme/next/_config.yml`文件来确认自己的NexT版本。

如果你的Hexo主题不是NexT，你可以参考[Hexo的NexT主题个性化：添加文章阅读量][教程]，这篇文章提供了一个通用的方法，理论上可以为Hexo博客的所有主题提供访客统计的功能。

# 配置LeanCloud

打开[LeanCloud官网][leancloud]，注册并完成邮箱激活后，进入控制台，创建一个新应用。

![][创建新应用]

进入我们新建的应用，进入`存储`分页，创建名为`Counter`的Class。

![][创建Class]

![][创建Class-2]

> **注意**：在这里，我们创建的Class名字必须为`Counter`，因为NexT会固定访问名字为`Counter`的Class，这个名字应该可以通过修改配置来实现更改，但是目前我找不到修改的方法。

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

## 设置安全域名
有时候我们的会在本地通过`locahost:4000`浏览并编辑我们的页面，在这种情况下，LeanClound会记录很多没有意义的浏览次数。为了让统计的浏览次数有意义，我们可以在`应用->设置->安全中心->Web安全域名`中设置自己博客的域名，只有该域名可以访问LeanCloud系统，因此只会记录在这个域名下的访客数据。

![][设置安全域名]

完成上面所有的设置后，重新部署你的博客，应该可以看到对应的效果。

# 参考资料
> [Hexo的NexT主题个性化：添加文章阅读量][教程]

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。

[教程]: http://www.jeyzhang.com/hexo-next-add-post-views.html
[leancloud]: https://leancloud.cn

[最终效果图片]: /uploads/hexo-counter/final.png
[创建新应用]: /uploads/hexo-counter/new-app.png
[创建Class]: /uploads/hexo-counter/new-class.png
[创建Class-2]: /uploads/hexo-counter/new-class-2.png
[应用key的位置]: /uploads/hexo-counter/app-key-position.png
[设置安全域名]: /uploads/hexo-counter/set-host.png
[底部标签]: /uploads/hexo-counter/footer.png
