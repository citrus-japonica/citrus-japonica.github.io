---
layout: default
title: Articles (kr-ko)
---

<div id="articles">
  <h1>Articles</h1>
  <ul class="posts noList">
    {% for post in site.posts %}
      {% if post.url contains "-ko" %}
        <li>
          <span class="date">{{ post.date | date: "%Y-%m-%d" }}</span>
          <a href="{{ post.url }}">{{ post.title }}</a>
        </li>
      {% endif %}
    {% endfor %}
  </ul>
</div>
