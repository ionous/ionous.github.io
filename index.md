---
title: io's new dev blog
---
# {{ page.title }}

<ul class="posts">
{% for post in site.posts %}
<li><span>{{ post.date | date_to_string }}</span> » <a href="{{post.url}}">{{post.title}}</a></li>
{% endfor %}
</ul>

