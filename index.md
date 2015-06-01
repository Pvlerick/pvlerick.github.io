---
layout: page
title: Philippe
tagline: that's me
---
{% include JB/setup %}

{% for post in site.posts %}
  <h2>One, Two</h2>
  {% include post.html %}
{% endfor %}