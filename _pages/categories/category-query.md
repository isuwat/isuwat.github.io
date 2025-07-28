---
title: "query tips"
layout: archive
permalink: categories/query
author_profile: true
type: posts
sidebar:
    nav: "sidebar-category"
---

{% assign posts = site.categories.query %}
{% for post in posts %}
  {% include archive-single.html type=page.entries_layout %}
{% endfor %}