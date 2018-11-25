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
在命令行中设置user.name和user.email配置信息：
```
git config --global user.name "你的Github用户名"
git config --global user.email "你的Github注册邮箱"
```
生成ssh密钥文件：
```
ssh-keygen -t rsa -C "你的Github注册邮箱"
```
默认不需要设置密码，直接回车即可。
然后找到生成的.ssh文件夹中的id_rsa.pub密钥，将内容复制到Github的SSH Key中。
在git Bash中检测Github公钥是否设置成功：
```
ssh git@github.com
```
如出现以下信息则说明设置成功：
```
PS D:\Work\MyBlog> ssh git@github.com
PTY allocation request failed on channel 0
Hi crystalww! You've successfully authenticated, but GitHub does not provide shell access.
Connection to github.com closed.
```

### 安装Node.js
安装[Node.js](https://nodejs.org/en/download/)会默认安装npm。检测是否安装成功可分别输入：node -v 和 npm -v。

使用国内的源（比如淘宝NPM镜像）去替换官方的源以加快下载包的速度。<br>
`npm config set registry http://registry.npm.taobao.org/`

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

---

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

#### 使用本地图片
修改_config.yml
```
post_asset_folder: true
```
在站点的根目录下（MyBlog文件夹）执行：
```
npm install https://github.com/CodeFalling/hexo-asset-image --save
```
存放图片的目录名与文章名保持一致，例如
```
MacGesture2-Publish
├── apppicker.jpg
├── logo.jpg
└── rules.jpg
MacGesture2-Publish.md
```
使用 `![logo](logo.jpg)`就可以插入图片。

#### 添加评论功能
**1.使用来必力评论系统**
- 在[来必力官网](https://livere.com)注册一个账号，选择免费的city版本安装。 
- 复制其中的uid字段。
- 打开主题配置文件，定位到livere_uid字段，粘贴上刚刚复制的uid。
- 修改`source/tags/index.md`
加入`comments: false`，否则标签和分类里面也会加载评论

**2.使用valine**（可以匿名评论）
- 注册[LeanCloud](https://leancloud.cn/dashboard/login.html#/signup)账号
- 创建应用：填写应用名、勾选`开发版`
- 进入设置页面查看App ID和App Key
- 在leancloud存储中创建Class，命名为Counter。修改next/_config.yml中leancloud_visitors，配置阅读统计：
```
leancloud_visitors:
  enable: true 设置为true 默认为false
  app_id:  #你的App ID
  app_key:  #你的App Key
  Dependencies:  https://github.com/theme-next/hexo-leancloud-counter-security #设置依赖
  security: true #没有hexo-leancloud-counter-security插件，请将security设置为false
  betterPerformance: true #更好的性能
```
- 在leancloud存储中创建Class，命名为Comment。修改next/_config.yml中valine部分，配置评论设置：
```
valine:
  enable: true # 设置为true，默认为false
  appid:  # 将应用key的App ID设置在这里
  appkey: # 将应用key的App Key设置在这里
  notify: true# 邮箱通知 , https://github.com/xCss/Valine/wiki，默认为false
  verify: true# 验证码 默认为false
  placeholder: Just go go ^_^ # 初始化评论显示
  avatar: wavatar # 头像风格，默认为mm，可进入网址：https://valine.js.org/visitor.html查看头像设置
  guest_info: nick,mail,link # 自定义评论标题
  pageSize: 10 # 分页大小，10页就自动分页
  visitor: true # 是否允许游客评论 ，进入官网查看设置：https://valine.js.org/visitor.html
```

#### 设置侧边栏社交
编辑主题配置文件，定位到字段`social`，然后添加社交站点名称与地址即可。
```
# Social links
social:
  GitHub: https://github.com/crystalww || github
  ...
```

编辑主题配置文件，在`social_icons`字段下添加社交站点名称（注意大小写）与图标。
```
social_icons:
  enable: true
  icons_only: false
  transition: false
  GitHub: github
  Twitter: twitter
  Weibo: weibo
  Linkedin: linkedin
```
图标名称可以参考：http://fontawesome.io/cheatsheet/。

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