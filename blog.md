---
layout: page
title: "Blog"
---

# Blog

Welcome to my blog! Here you'll find posts on coding, tech tips, and more insights from my development journey.

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}
