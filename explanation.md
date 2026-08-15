---
title: Explanation
description: Understand why the arena is built this way.
permalink: /explanation/
---

# Explanation

Background and reasoning — why the protocol is this narrow, why harness differences show up the
way they do, and design decisions that don't fit a tutorial or reference page.

<div class="index-list">
{% assign ordered = site.explanation | sort: "order" %}
{% for p in ordered %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
