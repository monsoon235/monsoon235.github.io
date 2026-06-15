# [monsoon-cs.moe](https://monsoon-cs.moe/)

My personal blog, powered by [Hugo](https://gohugo.io/) and [PaperMod](https://github.com/adityatelange/hugo-PaperMod), and hosted on Cloudflare Pages.

## Develop

```sh
git clone --recurse-submodules <repo>
hugo server          # local preview at http://localhost:1313
hugo --gc --minify   # production build into ./public
```

Pinned Hugo version: see `HUGO_VERSION` in `.github/workflows/pages.yml`.
