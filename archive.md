---
layout: page
title: Archive
permalink: /archive/
---
{% for post in site.posts %}

{{ post.date | date_to_string }} » [ {{ post.title }} ]({{ site.baseurl }}{{ post.url }}) {% endfor %}

{% include archive.html %}