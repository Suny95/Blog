# Hexo-theme-diaspora

### 安装主题

``` bash
$ git clone https://github.com/Fechin/hexo-theme-diaspora.git diaspora
```


### 启用主题

修改Hexo配置文件 `_config.yml` 主题项设置为diaspora


``` yaml

...
theme: diaspora
...
```
### 新建文章模板

``` markdown
---
title: My awesome title
date: 2016-10-12 18:38:45
categories: 
    - 分类1
    - 分类2
tags: 
    - 标签1
    - 标签2
mp3: http://domain.com/awesome.mp3
cover: http://domain.com/awesome.jpg
---
```

### 创建分类页
1 新建一个页面，命名为 categories 。命令如下：
```
hexo new page categories
```
2 编辑刚新建的页面，将页面的类型设置为 categories
```
title: categories
date: 2014-12-22 12:39:04
type: "categories"
---
```
主题将自动为这个页面显示所有分类。

### 创建标签页
1 新建一个页面，命名为 tags 。命令如下：
```
hexo new page tags
```
2 编辑刚新建的页面，将页面的类型设置为 tags
```
title: tags
date: 2014-12-22 12:39:04
type: "tags"
---
```
主题将自动为这个页面显示所有标签。


### 主题配置
```yml
# 头部菜单，title: link
menu:
  Whoami: /whoami
  Github: https://github.com/Fechin
  Twitter: https://twitter.com/FechinLi
  分类: /categories
  归档: /archives
  标签云: /tags

# 是否显示目录
TOC: false

# 是否自动播放音乐
autoplay: false

# 默认音乐（随机播放）
mp3: 
    - http://link.hhtjim.com/163/425570952.mp3
    - http://link.hhtjim.com/163/425570952.mp3

# 首页封面图, 为空时取文章的cover作为封面
welcome_cover: # /img/welcome-cover.jpg

# 默认文章封面图
cover: /img/cover.jpg

# Gitalk 评论插件（https://github.com/gitalk/gitalk）
gitalk:
    # 是否自动展开评论框
    autoExpand: false
    # 应用编号
    clientID: ''
    # 应用秘钥
    clientSecret: ''
    # issue仓库名
    repo: ''
    # Github名
    owner: ''
    # Github名
    admin: ['']
    # Ensure uniqueness and length less than 50
    id: location.pathname
    # Facebook-like distraction free mode
    distractionFreeMode: false

# 网站关键字
keywords: Fechin

# 要使用google_analytics进行统计的话，这里需要配置ID
google_analytics: 

# 网站ico
favicon: /img/favicon.png

# rss文件
rss: atom.xml
```

