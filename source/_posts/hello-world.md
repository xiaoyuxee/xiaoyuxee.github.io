---
title: Hello World
date: 2016-01-10 11:44:08
tags: 
categories: 生活随笔
toc: true
---

国际惯例：`Hello World`

折腾了许久，从 [wordpress](https://wordpress.com/) 到 [jekyll](http://jekyllrb.com/) ，以及国人开发的 [farbox](https://www.farbox.com/)... 待最近静下来梳理思路时问自己：*为何折腾，有没有必要，初衷又是什么。*

屡清楚之后选择了 [hexo](https://hexo.io)。

<!-- more -->

##  初衷

*	记录生活、工作中的点点滴滴
    其实起因是工作中的一个小需求：有个页面需要反向代理到别的团队，鉴于业务统计原因，页面中的链接必须是我们的域名，同时别人的页面还可正常访问。
    于是就将这个`nginx`文本替换的工单提给了PE（由PE统一处理，我们没有权限... ），结果PE吭哧吭哧半天没搞定，对`substitutions_filter_module`不熟悉，最后被告知“*自己申请权限去改，改好了由他审批发布*”。

    本来就对`niginx`感兴趣，一直没有时间折腾，索性就折腾一把。自己看文档，拉代码，编译，安装，测试。整个过程还是还是蛮有意思的，也遇到了各种问题。之后就在想，能否将这些分析问题、解决问题的思路和方法记录一下，备案的同时也给别人某种程度上的参考。

*	适合程序员方式的记录
    之前在本地都在使用[Ulysess](http://www.ulyssesapp.com)，一款专注于写作的软件，支持`Markdown`、`Dropbox`同步、多种文档格式的输出等。

*   可托管于Github
    无须担心备份等问题。

*   极简
    记录，分享。包括呈现方式，抛弃华丽的外表，不忘初心，分享内容而非炫酷的各种Blog效果。

## 框架选择

最后选择[hexo](https://hexo.io)的原因也很简单，其官方定义

>   A fast, simple & powerful blog framework

*   速度
    基于Node.js，支持多进程，几百篇文章几秒搞定

*   简单
    ``` bash
    # 创建一条日志
    hexo n "Hell World"

    # 发布
    hexo g

    # 部署
    hexo d
    ```
    几条简单的命令就可完成全部过程。

*   易扩展
    已经有很多成熟的插件，比如同时部署到Github和Gitcafe等

## 模板选择

[maupassant](https://github.com/pagecho/maupassant)，简约但不简单。 官方定义：

> A simple template with great performance on different devices.

已支持的平台：[Typecho](https://github.com/pagecho/maupassant/), [Octopress](https://github.com/pagecho/mewpassant/), [Farbox](https://github.com/pagecho/Maupassant-farbox), [Wordpress](https://github.com/iMuFeng/maupassant), [Ghost]( https://github.com/LjxPrime/maupassant), [Hexo](https://github.com/tufu9441/maupassant-hexo)。

## 参考资料

*   [hexo你的博客](http://ibruce.info/2013/11/22/hexo-your-blog/)
*   [大道至简——Hexo简洁主题推荐](https://www.haomwei.com/technology/maupassant-hexo.html)
*   [NexT - an elegant theme for hexo](http://theme-next.iissnan.com/)


