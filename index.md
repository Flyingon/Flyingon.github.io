---
title: 记录分享成长
layout: post
home: true
---

欢迎回来。

这里主要记录后端开发、数据库、Go、工程工具和一些长期可复用的技术笔记。

## 这里写什么
- 把踩过的坑和排障过程沉淀下来，方便以后快速回查
- 把零散知识重新整理成专题，尽量让每篇文章都能独立阅读
- 继续维护那些真正有复用价值的内容，而不是只追一时热点

## 当前重点
- Go 工程与调试
- 数据库与缓存
- 计算机基础与网络
- 工具链和项目实践

## 最近更新
{% for post in site.posts limit: 8 %}
- [{{ post.title }}]({{ post.url }}) · {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

## 常用工具
- 在线 JSON: [https://www.bejson.com/](https://www.bejson.com/)
- Nginx 配置参考: [https://www.digitalocean.com/community/tools/nginx?global.app.lang=zhCN](https://www.digitalocean.com/community/tools/nginx?global.app.lang=zhCN)
- C++ 实时转汇编: [https://godbolt.org/](https://godbolt.org/)
- 代码执行可视化: [https://pythontutor.com/visualize.html#mode=edit](https://pythontutor.com/visualize.html#mode=edit)
- 算法可视化: [VisuAlgo](https://visualgo.net/)、[B+Tree](https://www.cs.usfca.edu/~galles/visualization/BPlusTree.html)

## 站点说明
- 博客基于: [Jekyll](https://jekyllrb.com/)
- 托管于: [GitHub Pages](https://pages.github.com)
- 代码样式选择:
  
pygmentize -f html -a .highlight -S solarized-light > pygments.css

继续向前走。
![image](/assets/img/index.jpg)
