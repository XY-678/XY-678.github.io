---
layout: default
title: 首页
---

# 欢迎来到我的技术博客

这里记录学习笔记、编程实践和有趣的技术探索。

[关于本站]({{ '/about/' | relative_url }}) · [GitHub](https://github.com/XY-678)

## 最新文章

{% if site.posts.size > 0 %}
{% for post in site.posts %}
### [{{ post.title }}]({{ post.url | relative_url }})

<small>{{ post.date | date: "%Y-%m-%d" }}</small>

{{ post.description | default: post.excerpt | strip_html | truncate: 120 }}

{% endfor %}
{% else %}
文章正在整理中，敬请期待。
{% endif %}

---

> 这是一个使用 GitHub Pages 和 Jekyll 构建的个人博客。
