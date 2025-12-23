---
layout: archive
title: "Cloud"
archive_type: category
category: cloud
permalink: /categories/cloud/
paginate: 5         # optional, controls posts per page
show_date: true      # optional
show_excerpt: true   # optional
---

{% assign posts = site.categories.cloud %}
{% for post in posts %}
  <div class="post-card">
    <h4><a href="{{ post.url }}">{{ post.title }}</a></h4>
    <p>{{ post.excerpt }}</p>
  </div>
{% endfor %}

