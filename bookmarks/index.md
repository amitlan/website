---
layout: default
title: Links
---
<a href="https://amitlan.github.io/writing">Writing</a> | <a href="https://amitlan.github.io/talks">Talks</a> | <a href="https://amitlan.github.io/photolog">Pictures</a> | <b>Links</b>
<hr>

<table cellspacing="15" class="posts">
  {% for post in site.tags["links"] %}
  <tr>
    <td><a href="{{ post.external_url }}"><b>{{ post.title }}</b></a></td><td>{{ post.description}}</td>
  </tr>
  {% endfor %}
</table>
