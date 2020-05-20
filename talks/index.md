---
layout: default
title: Talks
---
<table cellspacing="15" class="posts">
  {% for post in site.tags["talks"] %}
  <tr>
    <td><a href="{{ post.external_url }}"><b>{{ post.title }}</b></a></td><td>{{ post.description}}</td>
  </tr>
  {% endfor %}
</table>
