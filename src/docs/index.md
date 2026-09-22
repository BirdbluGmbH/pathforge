---
layout: default
is_doc: true
permalink: /docs/
base: "../"
title: Documentation
description: Install PathForge, get your first route, and go from a single agent to a crowd — the complete PathForge documentation.
---
# Documentation

Everything you need to install PathForge, get your first route, and take it from a
single agent to a crowd. Start with **Getting Started**; the **API Reference** has the
full method-by-method detail.

<div class="docs-list">
{% for d in site.data.docs %}
  <a href="{{ page.base }}{{ d.url }}"><span class="t">{{ d.title }}</span><span class="d">{{ d.desc }}</span></a>
{% endfor %}
</div>
