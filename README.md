jirihnidek.github.io
====================

[Jiri Hnidek](http://jirihnidek.github.io)'s personal pages using [Bootstrap](http://getbootstrap.com/), [Jekyll](http://jekyllrb.com/) and [GitHub Pages](https://pages.github.com/). Thus it is necessary to have installed [Ruby](https://www.ruby-lang.org/en/), [Node.js](http://nodejs.org/) and [Bundler](http://bundler.io/).

To run pages locally run following code:

```bash
git clone git@github.com:jirihnidek/jirihnidek.github.io.git
cd jirihnidek.github.io
bundle install
jekyll server --watch
```

Then open following link in your browser: http://localhost:4000

When you update the page, then you should run following commands:

```bash
cd jirihnidek.github.io
bundle update
jekyll server --watch
```

When you want to have live preview of your changes, then you have to run guard command in another terminal:

```bash
guard
```