---
layout: page
title: Blog
permalink: /blog/
---

<ul>
  {% for post in site.posts %}
    <li>
      <h3>
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h3>
      <p class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</p>
      
      <p>{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
    </li>
    <hr>
  {% endfor %}
</ul>