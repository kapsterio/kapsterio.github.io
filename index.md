---
layout: page
title: Hello World!
tagline: Supporting tagline
---
<h1 class="page-heading">Posts</h1>

<ul class="post-list">
    {% for post in site.posts %}
      <li>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>

        <h2>
          <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        </h2>
      </li>
      <li>
          {{post.content | split:'<!--more-->' | first}}
          <h7><a href='{{post.url}}'>read more</h7>
      </li>
    {% endfor %}
</ul>