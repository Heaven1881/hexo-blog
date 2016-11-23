---
title: Hexo博客-NexT主题：使用不蒜子进行页面访客统计
tags:
  - hexo
  - next
  - 访客统计
  - 不蒜子
categories: Hexo博客-NexT主题
date: 2016-11-13 15:00:35
---


关于不蒜子统计，可以参考[http://ibruce.info/2015/04/04/busuanzi/](http://ibruce.info/2015/04/04/busuanzi/)，现在，如果你使用的博客是Hexo并且主题是NexT，那么你可以很方便的进行不蒜子的访客统计设置。

<!-- more -->

> *正在使用Hexo和NexT搭建你的博客吗？[点击这里](/categories/Hexo博客-NexT主题/)查看这个分类的更多文章。*

# 最终效果
- 在主页的底部显示PV数和UV数

![](/uploads/busuanzi-footer.png)

- 在每篇文章的头部显示PV数

![](/uploads/busuanzi-page.png)

> 对于一个页面来说，**PV**指的是这个页面被访问的次数，每次打开这个页面，**PV**数就会加一，而**UV**指的是访问这个页面的用户数，同一个用户重复打开这个页面并不会增加**UV**的数值

# 注意事项

在使用这篇博客之前，你需要先确保自己使用的Hexo博客的NexT主题。旧版的NexT主题可能不支持这篇文章提到的功能，在进行进一步的操作之前，确保自己使用的NexT版本支持对应功能。在这里，我使用的版本为`5.0.1`，你可以通过查看`/theme/next/_config.yml`文件来确认自己的NexT版本。

# 开始
打开`/theme/next/_config.yml`，找到如下的配置项

```yaml
# Show PV/UV of the website/page with busuanzi.
# Get more information on http://ibruce.info/2015/04/04/busuanzi/
busuanzi_count:
  # count values only if the other configs are false
  enable: false
  # custom uv span for the whole site
  site_uv: true
  site_uv_header: <i class="fa fa-user"></i>
  site_uv_footer:
  # custom pv span for the whole site
  site_pv: true
  site_pv_header: <i class="fa fa-eye"></i>
  site_pv_footer:
  # custom pv span for one page only
  page_pv: true
  page_pv_header: <i class="fa fa-file-o"></i>
  page_pv_footer:
  
```

将`enable`的值由`false`修改为`true`后，重新部署即可看到效果。当然，你还可以自定义上面的配置：

- `site_uv`表示是否显示整个网站的**UV**数
- `site_pv`表示是否显示整个网站的**PV**数
- `page_pv`表示是否显示每个页面的**PV**数

在你完成部署后，可能你会发现，自己网站的**PV**数和**UV**数都非常大，这是正常情况，因为使用不蒜子统计的用户都使用同一个存储空间，如果你的URL和别人重复，就会出现数据量异常。这样的情况一般出现在你使用`localhost:4000`访问自己在本地部署的网页的时候。如果你希望自己博客可以使用自己独立的一个存储空间，可以参考我的另一篇文章[Hexo博客-NexT主题：使用leancloud进行页面访客统计](/2016/05/31/add-count-for-hexo-next/)

---
> 本文欢迎转载，但是希望注明出处并给出原文链接。
> 如果你有任何疑问，欢迎在下方评论区留言，我会尽快答复。
> 如果你喜欢或者不喜欢这篇文章，欢迎你发邮件到[winton.luo@outlook.com](mailto:winton.luo@outlook.com)告诉我你的想法，你的建议对我非常重要。
