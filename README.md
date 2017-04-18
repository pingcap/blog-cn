# blog-cn

## website blog 规范

### .md 文件要以英文命名

### meta 信息

头部的 meta 信息必须包含一下内容：

- title 文章标题
- author 文章作者
- date 文章发布日期 *格式:yyyy-mm-dd*
- summary 文章简介
- tags 标签分类 多个tags之间用空格分开

```
---
title: Blog Title
author: Author
date: yyyy-mm-dd
summary: Blog Summary
tags: Tag1 Tag2
---
```


### 文档内容

- 正文中不需要将 blog 的标题写在最前面 请将标题统一写在 meta 信息中 方便html中使用统一样式
- 正文中用到的图片请统一放在 blog 或 blog-cn repo 的 media 目录下
- 正文中引用的图片命名避免用1、2、3之类的 避免冲突