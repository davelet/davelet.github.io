---
layout: page
title: 站不端正的蜡烛，泪多命短
permalink: /blog/
---
{% for post in site.posts %}
<article class="post">

    <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>
    {{post.date}} 

</article>
{% endfor %}