---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: Y.A.A.P.B.
permalink: /
---

Welcome!

This is yet another "another personal blog" (Y.A.A.P.B.)
Nothing special, I just hope to provide some stories about my engineering career, in a post-LLM era.

---

{% for post in site.posts %}
[{{ post.date | date: "%Y-%m-%d" }}] [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
