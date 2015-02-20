---
layout: page
title: A Blog
---
{% include JB/setup %}

<h4  >Recent Posts</h4  >

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a><br  /> <i  >{{post.description}}</i  ></li>
  {% endfor %}
</ul>
