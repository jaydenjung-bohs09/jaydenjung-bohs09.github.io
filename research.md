---
layout: single
title: "Research"
permalink: /research/
sidebar:
  nav: "main"
---

## Research
Explore completed research frameworks below.

{% for post in site.research reversed %}
<div style="margin: 1rem 0;">
  <a href="{{ post.url | relative_url }}" class="btn btn--primary" style="text-decoration:none;">
    {{ post.title }}
  </a>
</div>
{% endfor %}
