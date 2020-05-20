---
layout: default
title: Links
---
<ul class="posts">
  {% for post in site.tags["links"] %}
    <li>
      <a href="{{ post.external_url }}">{{ post.title }}</a>
      <div class="text" style="color: #a0aec0">{{ post.description }}</div>
    </li>
  {% endfor %}
</ul>
