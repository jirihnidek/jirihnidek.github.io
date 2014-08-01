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

{% endfor %}
```

## Tutorials

* https://www.andrewmunsell.com/blog/ultimate-jekyll-tutorial
* http://code.tutsplus.com/articles/building-static-sites-with-jekyll--net-22211
* http://joshualande.com/jekyll-github-pages-poole/
