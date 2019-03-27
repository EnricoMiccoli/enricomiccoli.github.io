---
title: Home V&D
description: descrizione della pagine titolata pippo
layout: default
---
# Vars and Dirs
This is my blog, where I document the math-y, programming-y things I've done.

## Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">
        {{ post.date | date: "%Y-%m-%d" }}
      </a>
      {{ post.title }}
    </li>
  {% endfor %}
</ul>

