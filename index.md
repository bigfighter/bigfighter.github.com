---
layout: page
title: Yet Another Home Automation Project
---

## Table of Contents

<ul class="posts">
	{% for post in site.posts %}
		<li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
	{% endfor %}
</ul>

## Links

[Jekyll Quick Start](http://jekyllbootstrap.com/usage/jekyll-quick-start.html)
