---
layout: default
title: Blog
---

# Blog

<ul>
    {% for post in site.posts %}
        <li>
            <a href="{{ post.url }}">{{ post.title }}</a>
                <small><i>{{ post.date | date: "%B %-d, %Y"}}</i></small>
        </li>
    {% endfor %}
</ul>