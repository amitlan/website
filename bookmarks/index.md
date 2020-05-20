---
layout: default
title: Links
---
<table cellspacing="15" class="posts">
  {% for post in site.tags["links"] %}
  <tr>
    <td><a href="{{ post.external_url }}"><b>{{ post.title }}</b></a></td><td>{{ post.description}}</td>
  </tr>
  {% endfor %}
</table>
