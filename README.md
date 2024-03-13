# readme

## Running locally:
1. https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll

```
cd docs
bundle install
bundle exec jekyll serve
```

Edit the CSS by changing [`main.scss`](docs/assets/main.scss).

## Things we use:
1. Theme: https://github.com/yous/whiteglass
1. Linkify `<h1>`, `<h2>`, etc tags: https://github.com/allejo/jekyll-anchor-headings

## Known issues
The site doesn't work via the url https://www.dasl.cc. This is because my username [ends with a hyphen](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/troubleshooting-custom-domains-and-github-pages#url-formatting-on-linux). But it works for all other combinations of the url, e.g. http://www.dasl.cc or https://dasl.cc.