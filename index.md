---
layout: default
title: io's new dev blog
---
# {{ page.title }}

<ul class="posts">
{% for post in site.posts %}
<li><span>{{ post.date | date_to_string }}</span> » [{{post.title}}]({{post.url}})</li>
{% endfor %}
</ul>

