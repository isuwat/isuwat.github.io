---
title: "WEB 개발 tips"
layout: archive
permalink: categories/web
author_profile: true
type: posts
sidebar:
    nav: "sidebar-category"

---

{% assign posts = site.categories.web %}
{% for post in posts %}
  {% include archive-single.html type=page.entries_layout %}
{% endfor %}