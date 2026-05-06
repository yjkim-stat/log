---
layout: page
permalink: /explorations/
title: explorations
nav: true
nav_order: 3
description: Short writeups introducing recent ML techniques together with my own hands-on experiments.
---

{% assign sorted = site.explorations | sort: 'date' | reverse %}
<ul class="post-list">
  {% for e in sorted %}
    <li>
      <h3><a href="{{ e.url | relative_url }}">{{ e.title }}</a></h3>
      <p class="post-meta">
        {{ e.date | date: "%b %d, %Y" }}
        {% if e.tags %} · {{ e.tags | join: ", " }}{% endif %}
      </p>
      {% if e.description %}<p>{{ e.description }}</p>{% endif %}
    </li>
  {% endfor %}
</ul>
