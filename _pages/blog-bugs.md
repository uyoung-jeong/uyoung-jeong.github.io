---
title: "Bugs"
layout: archive
permalink: blog/bugs/
author_profile: true
types: posts
sidebar:
    nav: "sidebar-category"
---

{% assign posts = site.categories.bugs %}
{% for post in posts %} {% include archive-single.html type=page.entries_layout %} {% endfor %}
