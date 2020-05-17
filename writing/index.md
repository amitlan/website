---
layout: default
title: Writing
---
<b>Writing</b> | <a href="https://amitlan.github.io/talks">Talks</a> | <a href="https://amitlan.github.io/photolog">Pictures</a> | <a href="https://amitlan.github.io/bookmarks">Links</a>
<hr>
<table cellspacing="15" class="posts">
  {% for post in site.posts %}
  <tr>
    <td><a href="{{ post.url }}"><b>{{ post.title }}</b></a></td><td><div class="publish-date"><time pubdate="">{{ post.date | date: "%B %-d, %Y" }}</time></div></td>
  </tr>
  {% endfor %}
</table>
