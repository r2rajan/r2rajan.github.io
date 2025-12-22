---
layout: home
title: "Ramesh Rajan | Cloud, Security & AI Insights"
permalink: /
---

# Welcome to My Blog

Hi, I’m **Ramesh Rajan**, sharing insights at the intersection of **Cloud, Security, and AI**.  
I focus on **strategic guidance, technical deep-dives, and thought leadership** to help professionals navigate modern technology landscapes.

Connect with me on [GitHub](https://github.com/r2rajan) or [LinkedIn](https://linkedin.com/in/rrajan).

---

## Explore Topics

<div class="category-cards">

<div class="card">
  <h3>Cloud</h3>
  <p>Articles on **AWS, serverless architectures, cloud strategies, and modern infrastructure design**.</p>
  <a href="/categories/cloud/">Explore Cloud →</a>
</div>

<div class="card">
  <h3>Security</h3>
  <p>Insights into **security architectures, governance, and cloud security strategies**.</p>
  <a href="/categories/security/">Explore Security →</a>
</div>

<div class="card">
  <h3>AI</h3>
  <p>Posts on **AI/ML technologies, LLM applications, and integrating AI into cloud solutions**.</p>
  <a href="/categories/ai/">Explore AI →</a>
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
