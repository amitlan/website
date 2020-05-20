---
layout: default
title: Talks
---
<ul class="posts">
  {% for post in site.tags["talks"] %}
    <li>
      <a href="{{ post.external_url }}">{{ post.title }}</a>
      <div class="text">{{ post.description }}</div>
    </li>
  {% endfor %}
</ul>
