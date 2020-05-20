---
layout: default
title: Blog
---
<ul class="posts">
  {% for post in site.tags["writing"] %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <div class="publish-date" style="color: #718096"><time pubdate="">{{ post.date | date: "%B %-d, %Y" }}</time></div>
    </li>
  {% endfor %}
</ul>
