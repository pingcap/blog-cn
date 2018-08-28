---
draft: true
---

# blog-cn

## 文档规范

### meta 信息

头部的 meta 信息可包含以下内容：

- `title` 文章标题
- `author` 文章作者, 格式: `['author-1', 'author-2']`
- `date` 文章发布日期, 格式: `yyyy-mm-dd`
- `summary` 文章简介
- `tags` 标签分类, 格式: `['tag-1', 'tag-2']`
- `category` 表示 blog 的类型，目前只有案例文章需要显示定义 `category: case`
- `url` 表示本篇文章在 PingCAP 官网中指定的 `url`， 而不是 HUGO 生成器默认生成的 `/blog-cn/filename/` 格式的 `url`
- `aliases` 表示可跳转到本篇文章在 PingCAP 官网中相应页面的 url list

其中 `title` `author` 是 **必填项**

```yml
---
title: Blog Title
author: ['Author']
date: yyyy-mm-dd
summary: Blog Summary
tags: ['Tag1', 'Tag2']
category: case
url: /the-website-case-url/specified-in-this-article/
aliases: ['/the-website-url/redirected-to-this-article']
---
```

### 文档内容

- 正文中用到的图片请统一放在 `blog` 或 `blog-cn` repository 的 `media` 目录下
- 正文中引用的图片命名避免用 1、2、3 之类的 避免冲突
