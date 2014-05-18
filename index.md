---
layout: page
title: The third man's blog
---
{% include JB/setup %}

{% for post in site.posts offset: 0 limit: 75 %}
<div class="row-fluid">
      <h1><strong><a href="{{ post.url }}">{{ post.title }}</a></strong></h4>
      <p>
        {{ post.content }}
      <p>
    <hr>
</div>
{% endfor %}
