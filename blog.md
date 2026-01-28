---
layout: page
title: Blog
permalink: /blog/
---

<div class="posts-wrapper">
  {% for post in site.posts %}
    <article class="post-card">
      <div class="post-card-header">
        <span class="post-meta-date">{{ post.date | date: "%Y-%m-%d" }}</span>
      </div>
      
      <h3 class="post-card-title">
        <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      </h3>

      <p class="post-card-excerpt">
        {{ post.excerpt | strip_html | truncatewords: 35 }}
      </p>

      <a href="{{ post.url | relative_url }}" class="read-more-link">Read Article →</a>
    </article>
  {% endfor %}
</div>