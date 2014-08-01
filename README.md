jirihnidek.github.io
====================

Jiri Hnidek's personal pages

TODO in index.html:

```
{% for post in site.posts %}

<div class="row-fluid">
  <div class="col-sm-12">
    <h3>{{ post.title }}</h3>
    <h4>{{ post.date | date_to_long_string }}</h4>
    <p>{{ post.excerpt }}</p>
    <p>
      <a href="{{ post.url }}">Read Rest</a>
    </p>
  </div>
</div>

TODO: post.html template

```
{% include header.html %}

<div class="row-fluid">
  <div class="col-sm-8">
    <h1>{{ page.title }}</h1>
    <p class="muted">{{ page.date | date_to_long_string }}</p>

    {{ content }}
  </div>
  <div class="col-sm-4">
    {% include sidebar.html %}
  </div>
</div>

{% include footer.html %}
```

{% endfor %}
```

## Tutorials

* https://www.andrewmunsell.com/blog/ultimate-jekyll-tutorial
* http://code.tutsplus.com/articles/building-static-sites-with-jekyll--net-22211
* http://joshualande.com/jekyll-github-pages-poole/
