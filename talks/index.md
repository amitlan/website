---
layout: default
title: Talks
---
<a href="https://amitlan.github.io/writing">Writing</a> | <b>Talks</b> | <a href="https://amitlan.github.io/photolog">Pictures</a> | <a href="https://amitlan.github.io/bookmarks">Links</a>
<hr>
<table cellspacing="15" class="posts">
  {% for post in site.tags["talks"] %}
  <tr>
    <td><a href="{{ post.external_url }}"><b>{{ post.title }}</b></a></td><td>{{ post.description}}</td>
  </tr>
  {% endfor %}
</table>
