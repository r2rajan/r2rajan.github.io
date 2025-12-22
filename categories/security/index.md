---
layout: archive
title: "Security"
archive_type: category
category: security
permalink: /categories/security/
paginate: 5         # optional, controls posts per page
show_date: true      # optional
show_excerpt: true   # optional
---

{% assign posts = site.categories.security %}
{% for post in posts %}
  <div class="post-card">
    <h4><a href="{{ post.url }}">{{ post.title }}</a></h4>
    <p>{{ post.excerpt }}</p>
  </div>
{% endfor %}

{% for post in site.categories.security %}
  <p>{{ post.title }}</p>
{% endfor %}
