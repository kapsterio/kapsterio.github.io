---
layout: "default"
title: how to use draft
author: "johnny"
---
test how draft work!
![My helpful screenshot]({{ site.url }}/assets/1.jpg)
{% for file in site.static-files %}
<li>
 {{file.path}}
 {{file.extname}}
</li>
{% endfor %}
