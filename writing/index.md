---
layout: default
title: Blog
---
<table cellspacing="15" class="posts">
  {% for post in site.tags["writing"] %}
  <tr>
    <td><a href="{{ post.url }}"><b>{{ post.title }}</b></a></td><td><div class="publish-date"><time pubdate="">{{ post.date | date: "%B %-d, %Y" }}</time></div></td>
  </tr>
  {% endfor %}
</table>
<hr>
<ul class="posts">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <div class="publish-date"><time pubdate="">{{ post.date | date: "%B %-d, %Y" }}</time></div>
    </li>
  {% endfor %}
</ul>
