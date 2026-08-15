---
title: How-to guides
description: Accomplish a specific task.
permalink: /how-to/
---

# How-to guides

Goal-oriented steps for a specific task, assuming you already know the basics.

<div class="index-list">
{% assign ordered = site.how-to | sort: "order" %}
{% for p in ordered %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
