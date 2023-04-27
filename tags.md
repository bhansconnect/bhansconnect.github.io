---
layout: page
title: Tags
permalink: /tags/
---

{%- assign tags = site.tags | sort -%}
{%- for tag in tags -%}

{%- assign tagName = tag[0] -%}
{%- assign tagNumPosts = tag[1] | size -%}
<h3 id="{{tagName | uri_escape | downcase }}">{{ tagName }} ({{ tagNumPosts }})</h3>

<ul>
    {% assign sorted_posts = tag[1] | reversed %}
    {% for post in sorted_posts %}
    <li>
        <a href="{{ post.url }}">{{ post.title }}</a> -
        <time datetime="{{ post.date | date_to_xmlschema }}"
              itemprop="datePublished">{{ post.date | date: "%b %-d, %Y" }}</time>
    </li>
    {% endfor %}
</ul>

{%- endfor -%}