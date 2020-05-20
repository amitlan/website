---
layout: default
title: Photos
---
<ul class="posts">
  {% for post in site.tags["pics"] %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <div class="publish-date" style="color: #f7fafc"><time pubdate="">{{ post.date | date: "%B %-d, %Y" }}</time></div>
    </li>
  {% endfor %}
</ul>
