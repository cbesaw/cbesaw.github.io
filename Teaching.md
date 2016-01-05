---
layout: page
title: Teaching
---

## Blog Posts

{% for teaching in site.lessons %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endfor %}
