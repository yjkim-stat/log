---
layout: page
permalink: /reports/
title: reports
nav: true
nav_order: 5
description: Periodic analyses on recent inference architecture trends.
---

{% assign sorted = site.reports | sort: 'date' | reverse %}
<ul class="post-list">
  {% for r in sorted %}
    <li>
      <h3>
        {% if r.report_url %}
          <a href="{{ r.report_url | relative_url }}" target="_blank">{{ r.title }}</a>
        {% else %}
          <a href="{{ r.url | relative_url }}">{{ r.title }}</a>
        {% endif %}
      </h3>
      <p class="post-meta">
        Released {{ r.date | date: "%b %d, %Y" }}
        {% if r.tags %} · {{ r.tags | join: ", " }}{% endif %}
      </p>
      {% if r.description %}<p>{{ r.description }}</p>{% endif %}
    </li>
  {% endfor %}
</ul>
