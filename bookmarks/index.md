---
layout: default
title: Links
---
<h2>People</h2>
  {% for post in site.tags["people"] %}
    <li>
      <a href="{{ post.external_url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
<h2>Articles and Papers</h2>
<ul class="posts">
  {% for post in site.tags["links"] %}
    <li>
      <a href="{{ post.external_url }}">{{ post.title }}</a>
      <div class="text" style="color: #718096">{{ post.description }}</div>
    </li>
  {% endfor %}
</ul>
