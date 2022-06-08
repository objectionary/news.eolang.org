---
layout: static
title: "Contents"
date: 2017-03-02
permalink: /contents.html
description: |
  All articles published on the blog, in chronological
  order, starting from the most recent
keywords:
  - artipie
---

Total: {{ site.posts.size }}.

{% for post in site.posts %}
  <div>
    <div>
      <a href="{{ post.url }}">
        <span itemprop="name headline mainEntityOfPage">{{ post.title }}</span>
      </a>
    </div>
    <ul class="subline">
      <li>
        <time itemprop="author">
          <a href="https://github.com/{{ post.author }}">@{{ post.author }}</a>
        </time>
      </li>
      <li>
        <time itemprop="datePublished dateModified" datetime="{{ post.date | date_to_xmlschema }}">
          {{ post.date | date_to_string }}
        </time>
      </li>
      <li>
        <a href="{{ site.url }}{{ post.url }}#disqus_thread" itemprop="discussionUrl">comments</a>
      </li>
    </ul>
  </div>
{% endfor %}
