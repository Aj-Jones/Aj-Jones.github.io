---
layout: default
title: Retrospectives
permalink: /retrospectives
---

{% for post in site.posts %}
    {% if post.image %}
<a href="{{post.url}}">![]({{post.image}})</a>
    <br> 
    {% endif %}
{% endfor %}
