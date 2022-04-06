---
title: "Effective Java"
permalink: categories/effective-java
author_profile: true
---

{% assign posts = site.categories['Effective Java'] %}
{% for post in posts %} 
  {% include archive-single.html type=page.entries_layout %} 
{% endfor %}