# 写作指南

## 新建文章

```bash
hugo new content posts/my-new-post.md
```

或手动创建 `content/posts/文件名.md`。

图片统一放在 `static/images/` 目录下。

## 文章头部（Front Matter）

```yaml
---
title: 文章标题
date: 2026-05-12T16:00:00+08:00   # 注意带时区 +08:00
description: 一句话摘要，用于 SEO 和首页预览
tags:
  - 标签1
  - 标签2
draft: false                        # true = 草稿不发布
---
```

## 图片

图片放到 `static/images/`，文章里引用：

```markdown
![描述](/images/文件名.png)
```

默认显示宽度 300px。想自定义宽度用 HTML：

```html
<img src="/images/文件名.png" width="500">
```

## 常用语法

```markdown
**加粗** *斜体* ~~删除线~~ `行内代码`

## 二级标题
### 三级标题

[链接文字](https://example.com)

> 引用

- 无序列表
1. 有序列表

---
```

代码块：

````markdown
```bash
ping 1.1.1.1
```
````

表格：

```markdown
| 名称 | 端口 |
|------|------|
| BGP  | 179  |
```

## 发布

```bash
git add .
git commit -m "update"
git push
# 自动部署，1-2 分钟生效
```

## 本地预览

```bash
hugo server
# http://localhost:1313/
```
