# readme

## Running locally:
1. https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/testing-your-github-pages-site-locally-with-jekyll

Requires Ruby version 3.x, installed via `brew install ruby`.

```
cd docs
bundle install
bundle exec jekyll serve
```

Edit the CSS by changing [`main.scss`](docs/assets/main.scss).

## Creating a new post
Create a new post in the `docs/_posts` directory. Prefix the filename with the date. Note that if you use a future date, jekyll will not render the post on the blog homepage. I.e. if the current date is 2024-12-30 and you use the date 2024-12-31, the post will not show up on the homepage.

## Things we use:
1. Theme: https://github.com/yous/whiteglass
1. Linkify `<h1>`, `<h2>`, etc tags: https://github.com/allejo/jekyll-anchor-headings

## Previous issues
Until 2026-03, the site didn't work via the url https://www.dasl.cc. I suspect this is because my username [ends with a hyphen](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/troubleshooting-custom-domains-and-github-pages#url-formatting-on-linux). But it worked for all other combinations of the url, e.g. http://www.dasl.cc, http://dasl.cc, or https://dasl.cc.

As of 2026-03, I moved my DNS provider to cloudflare and enabled them to proxy requests to my site. That seemed to fix the problem with loading the site via https://www.dasl.cc ! I think it didn't work properly until I enabled them to proxy requests specifically.

<img width="1168" height="474" alt="Screenshot 2026-04-01 at 5 56 32 PM" src="https://github.com/user-attachments/assets/6fd5311e-3757-411e-ba04-0b480731b217" />
