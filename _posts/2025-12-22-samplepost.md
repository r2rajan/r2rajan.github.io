---
title: "Threat Modeling an AI Inference Pipeline on AWS"
excerpt: "A practical, security-first walkthrough of threat modeling a GenAI inference pipeline using AWS-native controls."
categories:
  - security
tags:
  - aws
  - genai
  - threat-modeling
  - iam
  - llm
layout: single
toc: true
toc_sticky: true
mermaid: true
sidebar:
  nav: categories
---

## Why Threat Modeling Matters for GenAI

Generative AI inference pipelines introduce **new attack surfaces** beyond traditional web applications:
- Prompt injection
- Model abuse and data exfiltration
- Over-privileged IAM roles
- Supply chain risks in model artifacts

A structured threat model helps identify and mitigate these risks **before** production deployment.

---

## Reference Architecture

The following architecture represents a common **serverless GenAI inference flow on AWS**.

<div class="mermaid">
graph TD
  User -->|HTTPS| CloudFront
  CloudFront --> WAF
  WAF --> API_Gateway
  API_Gateway --> Lambda
  Lambda -->|Invoke| Bedrock
  Lambda --> DynamoDB
</div>


