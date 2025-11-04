---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---
<section class="index-intro">
最近一直在写一些关于pg的文章，你可以从右上角的"PG文章"目录或者下边的全部文章列表中找到。
</section>
<hr/>

<section class="posts-grid">
  {% assign posts_list = paginator.posts | default: site.posts %}
  {% for post in posts_list %}
    <article class="post-card">
      <div class="post-body">
        <h3 class="post-title"><a href="{{ post.url | relative_url }}">❀ {{ post.title }}</a></h3>
        <time class="post-date">{{ post.date | date: "%Y-%m-%d" }}</time>
        <p class="post-excerpt">{{ post.excerpt | strip_html | truncate: 120 }}</p>
      </div>
    </article>
  {% endfor %}
</section>