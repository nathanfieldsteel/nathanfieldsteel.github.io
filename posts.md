---
layout: page
title: Posts
---

<ul>
  {% for post in site.posts %}
    <li>
       {{ post.date | date: "%Y-%m-%d" }}: <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
