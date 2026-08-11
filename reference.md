---
title: Reference
description: Look up an exact fact.
permalink: /reference/
---

# Reference

Precise, exhaustive facts — the move contract's exact fields, bounds, and scoring rules.

<div class="index-list">
{% for p in site.reference %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
