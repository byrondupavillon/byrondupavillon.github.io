---
layout: page
title: Cyber Security
icon: fas fa-shield-alt
order: 55
---

{% assign groups = site.data.cyber_security_links | group_by_exp: 'l', "l.section | default: 'General'" %}
{% for g in groups %}
### {{ g.name }}
<ul>
  {% assign items = g.items | sort_natural: "name" %}
  {% for link in items %}
  <li>
    <a href="{{ link.url }}" target="_blank" rel="noopener">{{ link.name }}</a>
    {% if link.desc %} — {{ link.desc }}{% endif %}
  </li>
  {% endfor %}
</ul>
{% endfor %}
