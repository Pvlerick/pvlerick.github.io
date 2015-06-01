---
layout: page
title: Philippe
tagline: that's me
---
{% include JB/setup %}

{% for post in paginator.posts %}
   <h2>{{ post.title }}</h2>
   <div class="date">{{post.date | date: "%A, %d %B %Y %H:%M:%S %Z" }}</div>
   <div>
    {{ post.content }}
   </div>
   <hr>
{% endfor %}