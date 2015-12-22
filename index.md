---
layout: page
---

<ul class="post-list">
    {% for post in site.posts %}
    
      <li>
        <h2>
          <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        </h2>
        <span class="post-meta">{{ post.date | date: "%b %-d, %Y" }}</span>
      </li>
      <li class="article">
          {{post.content | split:'<!--more-->' | first}}
          <h7><a href='{{post.url}}' class="more-link">read more...</h7>
      </li>
    
    {% endfor %}
</ul>