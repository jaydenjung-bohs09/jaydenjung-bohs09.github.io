---
layout: single
title: "Activities"
permalink: /activities/
sidebar:
  nav: "main"
---

## Activities
Explore completed activities below.

{% for post in site.research reversed %}
<div style="margin: 1rem 0;">
  <a href="{{ post.url | relative_url }}" class="btn btn--primary" style="text-decoration:none;">
    {{ post.title }}
  </a>
</div>
{% endfor %}
