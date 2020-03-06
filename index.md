---
title: io's occasional daily updates
---
# {{ page.title }}

<ul class="posts">
{% for post in site.posts %}
<li><span>{{ post.date | date_to_string }}</span> » <a href="{{post.url}}">{{post.title}}</a></li>
{% endfor %}
</ul>

Haven't had enough?
For even rarer, longer form posts check out:<br>
[http://dev.ionous.net/](http://dev.ionous.net/)