---
layout: default
title: 📚 Posts
---

<h1>{{ page.title }}</h1>

<ul>
  {% assign myposts = site.pages | where_exp: "page", "page.path contains 'posts/'" %}
  {% for post in myposts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
