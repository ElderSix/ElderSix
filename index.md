---
layout: page
title: 老六的休闲会所, 在这里写写自己的东西
tagline: Supporting tagline
---
{% include JB/setup %}

## 关于我

* 职业  
大龄软件工程师  

* 技术领域  
Linux OS  
IaaS层基础设施软件: 分布式存储, 软件定义网络  
并行计算

* 爱好  
游戏开发
绘画
    
## 文章列表

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

