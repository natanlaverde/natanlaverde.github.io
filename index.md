---
layout: page
title: Início
<!-- group: navigation -->
<!-- tagline: Postagens -->
---
{% include JB/setup %}


## Posts
<ul class="posts">
  {% for post in site.posts %}
  	{% assign date = post.date %}
    <li><span>{% include date_format %}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>



