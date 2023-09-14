---
layout: tag
title: "Tag: Cisco"
permalink: /tags/cisco
---

<ul class="post-list">
  {%- for post in site.tags["Cisco"] -%}
    <li>
      {%- assign date_format = "%-d %B %Y" -%}
      <span>
        {{ post.date | date: date_format }}
      </span>
      <h3>
        <a class="post-link" href="{{ post.url | relative_url }}">
          {{ post.title }}
        </a>
      </h3>
    </li>
  {%- endfor -%}
</ul>