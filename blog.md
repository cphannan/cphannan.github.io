# Blog

<ul class="post-list">
  {% for post in site.posts %}
    <li>
      {%- assign date_format = "%b %-d, %Y" -%}
      <span class="post-meta">
        {{ post.date | date: date_format }}
      </span>
    </li>
    <a href="{{ post.url }}">{{ post.title }}</a>
  {% endfor %}
</ul>