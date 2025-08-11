---
layout: page
title: Useful Links
icon: fas fa-link
order: 50
---

{% assign groups = site.data.useful_links | group_by_exp: 'l', "l.section | default: 'General'" %}
{% assign section_order = "HTTP Tools|Diff & Merge|Security|Learning Resources|General" | split: "|" %}

{% for section in section_order %}
  {% assign g = groups | where: "name", section | first %}
  {% if g %}
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
  {% endif %}
{% endfor %}

{%- comment -%}
Show any other sections not listed in section_order (fallback)
{%- endcomment -%}
{% for g in groups %}
  {% unless section_order contains g.name %}
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
  {% endunless %}
{% endfor %}
