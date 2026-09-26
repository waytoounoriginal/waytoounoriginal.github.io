---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: Y.A.A.P.B.
permalink: /
---

Welcome!

This is just another personal blog. I hope I will post engaging enough content.

{% for post in site.posts %}
[{{ post.date | date: "%Y-%m-%d" }}] [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
