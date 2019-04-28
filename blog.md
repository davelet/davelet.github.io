---
layout: page
title: Blog List
permalink: /blog/
---
{% for post in site.posts %}
<article class="post">

    <h1>{{post.date}} - <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

</article>
{% endfor %}