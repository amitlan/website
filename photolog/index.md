---
layout: default
title: Photos
---
<table cellspacing="15" class="posts">
  {% for post in site.tags["pics"] %}
  <tr>
    <td><a href="{{ post.url }}"><b>{{ post.title }}</b></a></td><td><div class="publish-date"><time pubdate="">{{ post.date | date: "%B %-d, %Y" }}</time></div></td>
  </tr>
  {% endfor %}
</table>
