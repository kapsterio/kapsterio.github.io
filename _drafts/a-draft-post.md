---
layout: "default"
title: how to use draft
author: "johnny"
fileurl: "http://127.0.0.1:4000"
---
test how draft work!
![My helpful screenshot]({{ page.fileurl }}/assets/1.jpg)
{% for file in site.static-files %}
<li>
 {{file.path}}
 {{file.extname}}
</li>
{% endfor %}

{% for sitepage in site.pages %}
 <li>
标题： {{sitepage.title}}
url： {{sitepage.url}}
ID： {{sitepage.id}}
path： {{sitepage.path}}
 </li>
{% endfor %}



 <li>
标题： {{page.title}}
url： {{page.url}}
ID： {{page.id}}
path： {{page.path}}
PRE: {{page.previous.title}}
NEXT: {{page.next.title}}
 </li>
