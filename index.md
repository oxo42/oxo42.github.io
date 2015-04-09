---
layout: page
title: John's Blog
---
{% include JB/setup %}

Posts
=====

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>


<p class="rss-subscribe">subscribe <a href="{{ "/rss.xml" | prepend: site.baseurl }}">via RSS</a></p>
