---
layout: home
title: "Ramesh Rajan | Cloud, Security & AI Insights"
permalink: /
show_sidebar: true
---
<!-- Hero Section -->
<div class="hero-banner">
  <img class="avatar" src="/assets/images/avatar.jpg" alt="Ramesh Rajan">
  <div class="hero-text">
    <h1>Ramesh Rajan</h1>
    <p>Strategic & Technical Insights on <strong> Cloud, Security, and AI</strong></p>
    <p>
      <a href="https://github.com/r2rajan">GitHub</a> |
      <a href="https://linkedin.com/in/rrajan">LinkedIn</a>
    </p>
  </div>
</div>
---


## Explore Topics

<div class="grid-row category-cards">

<div class="grid-item card">
  <h3>AI</h3>
  <p>Articles on **AWS, serverless architectures, cloud strategies, and modern infrastructure design**.</p>
  <a href="{{ '/categories/ai/' | relative_url }}">Explore Cloud →</a>
</div>

<div class="grid-item card">
  <h3>Security</h3>
  <p>Insights into **security architectures, governance, and cloud security strategies**.</p>
  <a href="{{ '/categories/security/' | relative_url }}">Explore Security →</a>
</div>

<div class="grid-item card">
  <h3>Cloud</h3>
  <p>Posts on **AI/ML technologies, LLM applications, and integrating AI into cloud solutions**.</p>
  <a href="{{ '/categories/cloud/' | relative_url }}">Explore AI →</a>
</div>

</div>

---

## Latest Posts

<div class="latest-posts">
{% for post in site.posts limit:5 %}
<div class="post-card">
  <h4><a href="{{ post.url }}">{{ post.title }}</a></h4>
  <p>{{ post.excerpt }}</p>
</div>
{% endfor %}
</div>

---

## Tools & Topics I Focus On

- **Cloud:** AWS Lambda, API Gateway, DynamoDB, serverless design  
- **Security:** IAM, governance, cloud security frameworks  
- **AI:** LLMs, Generative AI pipelines, AI integration in enterprise solutions
