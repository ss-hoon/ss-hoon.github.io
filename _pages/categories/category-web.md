---
title: "Web"
permalink: categories/web
author_profile: true
---

{% assign posts = site.categories['Web'] %}
{% for post in posts %} 
  {% include archive-single.html type=page.entries_layout %} 
{% endfor %}