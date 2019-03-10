---
layout: default
title: David's Software Blog
permalink: /blog/
---

# Blog posts

{% for post in site.posts -%}
- [{{ post.title }}]({{  post.url }})
{% endfor %}
