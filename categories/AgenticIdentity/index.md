---
layout: archive
title: "Agentic Identity"
archive_type: category
category: AgenticIdentity
permalink: /categories/AgenticIdentity/
paginate: 5         # optional, controls posts per page
show_date: true      # optional
show_excerpt: true   # optional
---

{% assign posts = site.categories["AgenticIdentity"] %}
{% for post in posts %}
  <div class="post-card">
    <h4><a href="{{ post.url }}">{{ post.title }}</a></h4>
    <p>{{ post.excerpt }}</p>
  </div>
{% endfor %}

