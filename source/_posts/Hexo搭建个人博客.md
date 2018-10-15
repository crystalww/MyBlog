---
title: Hexo搭建个人博客
date: 2018-10-15 16:46:14
categories:
- 其他
tags:
- Hexo
---
## 准备工作
### Github创建个人仓库
首先要有一个github账号，点击GitHub中的New repository创建新仓库，仓库名为：username.github.io。
<!-- more -->

### 安装git
在电脑上安装[Git客户端](https://git-scm.com/downloads)，并将Git与Github账号绑定。

### 安装Node.js
安装[Node.js](https://nodejs.org/en/download/)会默认安装npm。检测是否安装成功可分别输入：node -v 和 npm -v。

### 安装Hexo
```
npm install -g hexo-cli
```

### 创建MyBlog文件夹
```
hexo init
hexo generate 或 hexo g
hexo server 或 hexo s
```
在本地浏览器打开：http://localhost:4000


## 主题修改
### 下载主题

以hexo-next为例，在站点的根目录下（MyBlog文件夹）执行以下命令：
```
git clone https://github.com/iissnan/hexo-theme-next themes/next
```

### 启用主题
打开站点配置文件(_config.yml)，将themes字段的值修改为next

### 验证主题
```
hexo clean      # 清除 Hexo 的缓存
hexo s --debug  # 启动 Hexo 本地站点，并开启调试模式
```
使用浏览器访问 http://localhost:4000 ，检查站点是否正确运行。

### 个性化配置（参考[Next主题配置](http://theme-next.iissnan.com/getting-started.html)）

#### 选择Scheme
更改主题配置文件(themes/next/_config.yml)，搜索scheme字段，去掉需要启动的scheme前的注释即可。
```
# Schemes
#scheme: Muse
#scheme: Mist
scheme: Pisces
#scheme: Gemini
```

#### 修改站点配置文件
```
# Site
title: 标题
subtitle:
description:
keywords:
author: 作者
language: zh-Hans
timezone: Asia/Shanghai
```

#### 添加标签和分类
修改主题配置文件中的menu
```
menu:
  home: / || home
  #about: /about/ || user
  tags: /tags/ || tags
  categories: /categories/ || th
  archives: /archives/ || archive
  #schedule: /schedule/ || calendar
  #sitemap: /sitemap.xml || sitemap
  #commonweal: /404/ || heartbeat
```

**生成“分类”页并添加type属性**
```
hexo new page categories
```
成功后会提示
```
INFO  Created: ~/MyBlog/source/categories/index.md
```
根据上面的路径，找到index.md这个文件，修改为：
```
---
title: categories
date: 2018-10-15 11:17:11
type: "categories"
---
```
保存并关闭文件。

**给文章添加“categories”属性**
打开需要添加分类的文章，为其添加categories属性。hexo一篇文章只能属于一个分类，如果在“- 测试”下方添加“- xxx”，hexo会把分类嵌套（即该文章属于“- 测试”下的“- xxx”分类）。
```
title: Hexo搭建个人博客
categories:
- 测试  
```

**生成“标签”页并添加tpye属性**
```
hexo new page tags
```
成功后会提示：
```
INFO  Created: ~/MyBlog/source/tags/index.md
```
根据上面的路径，找到index.md这个文件，修改为：
```
---
title: tags
date: 2018-10-15 11:23:05
type: "tags"
---
```
保存并关闭文件。

**给文章添加“tags”属性**
打开需要添加标签的文章，为其添加tags属性。
```
title: Hexo搭建个人博客
categories:
- 测试  
tags:
- 博客
- hexo
```

#### 自定义hexo new生成md文件的选项
在/scaffolds/post.md文件中添加：
```
---
title: {{ title }}
date: {{ date }}
categories:
tags:
---
```

#### 修改文章底部带#号的标签
修改模板/themes/next/layout/_macro/post.swig，搜索 rel="tag">#，将#换成
```
<i class="fa fa-tags"></i>
```

#### 添加搜索功能
安装 hexo-generator-searchdb，在站点的根目录下执行以下命令：
```
npm install hexo-generator-searchdb --save
```

编辑站点配置文件，新增以下内容到末尾：
```
search:
  path: search.xml
  field: post
  format: html
  limit: 10000
```
编辑主题配置文件，启用本地搜索功能：
```
# Local search
local_search:
  enable: true
```
然后重新生成，查看：
```
$ hexo clean
$ hexo s -g
```

----

## 推送到Github上
修改站点配置文件：
```
deploy:
  type: git
  repository: https://github.com/Crystalww/crystalww.github.io.git
  branch: master
```
然后执行以下命令：
```
npm install hexo-deployer-git --save   # 安装git部署插件
hexo g      # 本地生成静态文件
hexo d      # 将本地静态文件推送至Github
```

完成后，打开浏览器，在地址栏输入你的放置个人网站的仓库路径，即 username.github.io。