# Posts

<div class="post-row">
{% for post in site.posts %}
  <div class="post-front">
    <img class="post-front__image" src="{{ post.image }}">
    <a class="preview-title" href="{{ post.url }}">{{ post.title }}</a>
    <div class="tag-group"> 
      {% for tag in post.tags %}
      <div class="tag"><span class="tag-text">{{ tag }}</span></div>
      {% endfor %}
    </div>
  </div>
  {% endfor %}
</div>