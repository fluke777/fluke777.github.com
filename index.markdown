---
layout: default
title: Tomas's scrapbook
---

{% for post in site.posts %} 
  <h3>{{post.date | date_to_string}} » <a href="{{ post.url }}">{{ post.title }}</a></h3>
  <div><em class="perex">{{ post.perex }}</em></div>
{% endfor %}
