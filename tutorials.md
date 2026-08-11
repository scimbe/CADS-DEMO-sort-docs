---
title: Tutorials
description: Learn by doing, start to finish.
permalink: /tutorials/
---

# Tutorials

Learn by doing — each one takes you from nothing to a real, working result, start to finish,
against the actual live arena at [sort.bunsenbrenner.org](https://sort.bunsenbrenner.org/).

<div class="index-list">
{% for p in site.tutorials %}
  <a class="index-item" href="{{ p.url | relative_url }}">
    <strong>{{ p.title }}</strong>
    {% if p.description %}<span>{{ p.description }}</span>{% endif %}
  </a>
{% endfor %}
</div>
