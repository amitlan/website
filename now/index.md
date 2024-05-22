---
layout: default
title: Now
---
<ul class="posts">
  {% for post in site.tags["now"] %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <div class="text" style="color: #718096">{{ post.description }}</div>
    </li>
  {% endfor %}
</ul>
