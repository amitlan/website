---
layout: default
title: Writing
---
<b>Writing</b> | <a href="https://amitlan.github.io/talks">Talks</a> | <a href="https://amitlan.github.io/photolog">Pictures</a> | <a href="https://amitlan.github.io/bookmarks">Links</a>
<hr>
Things I wrote down recently.

<ul class="posts">
  {% for post in site.posts %}
    <li>
      <div class="publish-date"><time pubdate="">{{ post.date | date: "%B %-d, %Y" }}</time></div>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
