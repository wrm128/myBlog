---
title: Hexo 添加订阅、搜索和站点地图
date: 2018-04-22 18:29:11
tags:
- Hexo
categories:
- 随笔
---

简单记录下如何给Hexo网站添加RSS订阅功能、站内搜索功能以及站点地图。这些功能都需要相应的插件支持(Hexo插件参见 https://hexo.io/plugins/), 然后修改配置文件，**最后需要重新部署**。

## 添加RSS订阅功能

1. 安装插件
`npm install hexo-generator-feed --save`
2. 在站点配置文件`_config.yml`中追加如下代码：
```
feed:
  type: atom
  path: atom.xml
  limit: 20
  hub:
  content:
  content_limit: 140
  content_limit_delim: ' '
```
- type - Feed type. (atom/rss2)
- path - Feed path. (Default: atom.xml/rss2.xml)
- limit - Maximum number of posts in the feed (Use 0 or false to show all posts)
- hub - URL of the PubSubHubbub hubs (Leave it empty if you don't use it)
- content - (optional) set to 'true' to include the contents of the entire post in the feed.
- content_limit - (optional) Default length of post content used in summary. Only used, if content setting is false and no custom post description present.
- content_limit_delim - (optional) If content_limit is used to shorten post contents, only cut at the last occurrence of this delimiter before reaching the character limit. Not used by default.

## 添加站内搜索功能

1. 安装插件
`npm install hexo-generator-search --save`
2. 在站点配置文件`_config.yml`中追加如下代码：
```
# Search站内搜索
search:
  path: search.xml
  field: post
```
- path - file path. By default is search.xml . If the file extension is .json, the output format will be JSON. Otherwise XML format file will be exported.
- field - the search scope you want to search, you can chose:
    - post (Default) - will only covers all the posts of your blog.
    - page - will only covers all the pages of your blog.
    - all - will covers all the posts and pages of your blog.

## 添加站点地图(Sitemap)
Sitemap 可方便网站管理员通知搜索引擎他们网站上有哪些可供抓取的网页。最简单的Sitemap形式，就是XML文件，在其中列出网站中的网址以及关于每个网址的其他元数据（上次更新的时间、更改的频率以及相对于网站上其他网址的重要程度为何等），以便搜索引擎可以更加智能地抓取网站。

简言之， Sitemap 对于搜索引擎优化(Search Engine Optimization，即SEO) 非常重要，在网站中加入 Sitemap 有利于搜索引擎的爬虫组件的抓取和收录网站内容。以下插件主要用于生成适用于谷歌和百度的sitemap文件。 

##### 1. 安装插件
```
npm install hexo-generator-sitemap --save // 生成sitemap.xml
hexo-generator-seo-friendly-sitemap --save // seo优化sitemap.xml
npm install hexo-generator-baidu-sitemap --save // 生成baidusitemap.xml
```

##### 2. 在站点配置文件`_config.yml`中追加如下代码：
```
# SiteMap
sitemap:
  path: sitemap.xml
baidusitemap:
  path: baidusitemap.xml
```

部署(`hexo g`)之后，如果public目录下生成了 sitemap.xml 和 baidusitemap.xml 就表示配置成功了。

##### 3. 验证网站，并提交sitemap文件
在我们向搜索引擎提交 sitemap 之前，搜索引擎需要先验证我们对网站的所有权。两个搜索引擎的验证入口分别为:
- [Google Search Console](https://www.google.com/webmasters/tools/home?hl=zh-CN)
- [百度站长平台](https://ziyuan.baidu.com/dashboard/index)

具体验证及提交方法请参考:
- [hexo 系列教程: (三) 搜索引擎收录](http://selfmaking.top/2017/01/01/hexo-%E7%B3%BB%E5%88%97%E6%95%99%E7%A8%8B-%E4%B8%89-%E6%90%9C%E7%B4%A2%E5%BC%95%E6%93%8E%E6%94%B6%E5%BD%95/) 
- [hexo干货系列: (六) hexo提交搜索引擎（百度+谷歌](http://tengj.top/2016/03/14/hexo6seo/)


