---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: yet-another-personal-blog
permalink: /
---

Welcome!

This is just another personal blog. I hope I will post engaging enough content.


{% for post in site.posts %}
  <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
  <small>{{ post.date | date: "%Y-%m-%d" }}</small>
  <p>{{ post.excerpt }}</p>
{% endfor %}
