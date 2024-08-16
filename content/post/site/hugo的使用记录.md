+++
title = "Hugo&Stack主题的使用记录"
description = "关于hugo日常使用的记录，包含归档、评论、搜索等"
date = 2024-08-10T16:32:55+08:00
draft = false
tags = ['hugo','stack']
categories = ["site"]
slug ='hugo_stack_record'
image = ""
+++

[hugo官网](https://gohugo.io/about/)|[Stack官网](https://stack.jimmycai.com/config/)

## 通过Github Action进行部署

[基于 Github Action 自动构建 Hugo 博客 -](https://www.lixueduan.com/posts/blog/01-github-action-deploy-hugo/)

[【Hugo网站搭建】GitHub Action自动化部署Hugo博客 | Eddy&#39;s blog](https://eddyblog.cn/posts/tech/github-action-hugo/)

## 标签云增加数量统计&归档页增加标签云:

[Hugo Stack主题装修笔记](https://thirdshire.com/hugo-stack-renovation/#%E5%9C%A8%E5%BD%92%E6%A1%A3%E9%A1%B5%E5%A2%9E%E5%8A%A0%E6%A0%87%E7%AD%BE%E4%BA%91tags)

## 增加返回顶部按钮：

[添加一键返回顶部功能 | Dlog](https://chowray.netlify.app/posts/it%E5%B0%8F%E8%AE%B0/2021-01-06-%E4%B8%80%E9%94%AE%E8%BF%94%E5%9B%9E%E9%A1%B6%E9%83%A8/)

## 增加友链:

参考[Hugo Stack主题装修笔记Part 2](https://thirdshire.com/hugo-stack-renovation-part-two/#%E5%8F%8C%E6%A0%8F%E5%8F%8B%E6%83%85%E9%93%BE%E6%8E%A5%E5%B9%B6%E5%A2%9E%E5%8A%A0%E5%8D%9A%E4%B8%BB%E5%A4%87%E6%B3%A8)，结合自己的版本和主题略有修改

1. 我的links.html是放在 layouts/_default下才有效，在layouts/page下面没起效

2. data/json中指定的图片路径按博客所说需要放置在assets下再挂载到static中，我直接放在了static/link-img下

3. 菜单-友链需要格外的配置，这里的路径是你对应的文件，我这里不是新建了一个friend文件夹，然后在下面再创建index.md,而是在post下直接创建了个friend.md
   
   ```yaml
           - identifier: friends
             name: 友链|Friends
             url: /post/friends
             weight: 2
             params:
               icon: friends
               newTab: true
   ```
   
   ![](http://picgo.qisiii.asia/post/1723690756125.jpg)

## URL：

### slug:

一个正常的网址格式是下面这样的，其中路径的最后一段就是slug。

```
 <协议>://<主机名>/<路径>/<文件slug>?<查询字符串>#<片段>`
```

据说简短清晰的slug有助于SEO，但哪怕没有这个功能，我们的文件名很多时候也是中文的，经过编码后可读性太差。因此，设置一个简短的slug是很有必要的。

建议在模板文件archetypes/default.md直接新增slug字段，这样就不用每个文件手动添加了

```toml
+++
title = "{{ replace .Name "-" " " | title }}"
description = ""
date = {{ .Date }}
image = ""
draft = true
slug = "{{ replace .Name "-" " " | title }}"
+++
```

### permalinks:

如果只设置slug，stack主题配置默认路径是 域名/p/slug这样子，其实对于文件的组织管理和展示路径依然不是很明朗，因此，我这里是以文件夹层级的方式生成路径。

在总配置中，修改或者新增

```
permalinks:
    post: /:sections/:slug
```

这里的:sections是指多个路径，但是hugo默认处理顶级目录会自动识别为section外，其他目录需要手动添加_index.md文件才行。

## 评论：

hugo支持许多评论框架，具体可以看[hugo-comments](https://gohugo.io/content-management/comments/)|[Comments | Stack](https://stack.jimmycai.com/config/comments)，我选择的是

- [Twikoo](https://twikoo.js.org/)

建议跟着[快速上手 | Twikoo 文档](https://twikoo.js.org/quick-start.html)去部署一遍，我这里选用的是[Hugging Face](https://twikoo.js.org/backend.html#hugging-face-%E9%83%A8%E7%BD%B2)免费部署的方案。

部署主要有两部分：一部分是生成一个mogodb的密钥串，二部分则是在hugging face上新建空间和项目，最终获得envId

然后修改站点配置文件

```
     comments:
        enabled: true
        provider: twikoo
     twikoo:
        envId: https://你的项目空间.hf.space
        region:
        path:
        lang:
```

## 小部件：

[Widgets | Stack](https://stack.jimmycai.com/config/widgets)

```yaml
widgets:
        homepage:
            - type: search
            - type: archives
              params:
                  limit: 5
            - type: categories
              params:
                  limit: 10
            - type: tag-cloud
              params:
                  limit: 10
```

在stack的配置文件中，搜索工具、归档工具、分类和标签都是启用的。但是启用并不代表默认会出现，依然需要各种地方指定了值才能出现。

### 分类&标签

在你的内容文件的front matter中，需要指定tag的值和categories的值，我这里的front matter是toml格式

```toml
tag = ['hugo','stack']
categories = ["个人网站"]
```

### 归档：

需要在content/post目录下新建archives.md，内容如下，不需要任何正文内容

```toml
+++
title = 'Archives'
layout = "archives"
hidden = true
comments = false
+++
```

layout是指定为分类样式，hidden表示该文件不会被展示出来，comments表示评论区关闭.

其次，需要在 项目/layouts/_default 或者 主题/layouts/_default下存在archives.html，由于stack主题下默认就有这个文件，所以正常来说添加了archives.md就可以使用归档功能了。

### 搜索:

#### 增加search.md

首先是同归档类似，需要在content/post目录下新建search.md，内容如下，不需要任何正文内容

```toml
+++
title = "search"
layout = "search"
hidden = true
comments = false
+++
```

#### 增加搜索引擎的文件

静态网站的搜索本质上是将你的文档进行分词汇总到一个json文件，实际搜索的时候就是在搜索这个json文件。比较常用的是[fuse.js](https://www.fusejs.io/),[lunr.js](https://lunrjs.com/)或者主题自己封装的搜索能力。

1. stack主题
   
   增加search.html和search.json
   
   需要在 项目/layouts/_default 或者 主题/layouts/_default 存在两个文件（对于stack来说），分别是search.html和search.json。
   
   在我clone的stack版本中，这两个文件在_default并不存在，但是在/layouts/page下面是有的，所以从cp layouts/page/* layouts/_default
   
   成功的话，会在public/post/search目录下生成index.html index.json两个文件，主要是index.json文件

2. fuse
   
   我有看过两个文章是使用fuse的，但是我自己都没有实验成功，加上对hugo的模板还不熟，先不折腾了。
   
   [给hugo添加搜索功能 | 搜百谷](https://sobaigu.com/hugo-set-featuer-search.html)
   
   https://ttys3.dev/blog/hugo-fast-search

3. Docsy主题
   
   可以参考官方文档[Search | Docsy](https://www.docsy.dev/docs/adding-content/search/#algolia-docsearch)

#### 修改总配置hugo.yaml

```yaml
outputs:
  home:
    - HTML
    - JSON
  page:
    - HTML
    - JSON
```

这里home和page两种kind都配置的原因是:

假如你没有修改过总配置中的permalinks，那么你的博客的路径应该是 域名/p/文章名，这种情况下kind对应的是home类型

![](http://picgo.qisiii.asia/post/1723663347953.jpg)

但是假如你像我一下修改了permalinks，比如我改成了` post: /:sections/:slug`，这个是为了让我的博客地址以路径+文件名来显示，但是由于有了层级关系，里面的博客文章的kind就变成了page，因此在配置的时候，将home和page两个都指定是最好的

![](http://picgo.qisiii.asia/post/1723663319408.jpg)

说实话，我其实还是不能理解这个kind的设计~

## 网站收录：

bing [将 Github Pages 个人博客录入搜索引擎（以 Bing 为例） - RainbowC0 - 博客园](https://www.cnblogs.com/RainbowC0/p/18107581)

google [Google搜索收录Github Pages（Blog可以被Google搜索到） | Marvin's blog](https://winxuan.github.io/posts/blog-google-search-include/)

## 参考文档:

[Hugo 归档页面制作 - Cory](https://huweim.github.io/post/blog_hugo_%E5%BD%92%E6%A1%A3%E9%A1%B5%E9%9D%A2%E5%88%B6%E4%BD%9C/)
