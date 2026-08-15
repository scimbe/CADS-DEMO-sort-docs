---
title: Reference
description: Look up an exact fact.
permalink: /reference/
---

# Reference

Precise, exhaustive facts — the move contract's exact fields, bounds, and scoring rules.

<div class="index-list">
{% assign ordered = site.reference | sort: "order" %}
{% for p in ordered %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
