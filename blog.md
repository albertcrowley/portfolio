---
# layout: page  # Or your theme's default layout
title: Blog Posts & Technical Articles
permalink: /blog/
---

## List of All Published Articles
 
{% for post in site.posts %}
- **[{{ post.title }}]({{ post.url | relative_url }})**
  <br>
  <small>Published: {{ post.date | date: "%B %d, %Y" }}</small>
  {% endfor %}

