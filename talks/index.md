---
layout: default
title: Talks
---
<ul class="posts">
  {% for post in site.tags["talks"] %}
    <li>
      <a href="{{ post.external_url }}">{{ post.title }}</a>
      <div class="publish-date"><time pubdate="">{{ post.description }}</div>
    </li>
  {% endfor %}
</ul>
