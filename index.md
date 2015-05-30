---
layout: page
title: Philippe
tagline: that's me
---
{% include JB/setup %}

{% for post in paginator.posts %}
  {% include post.html %}
{% endfor %}