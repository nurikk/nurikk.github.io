# AI Notes

[![Gem Version](https://img.shields.io/gem/v/jekyll-theme-chirpy)][gem]&nbsp;
[![GitHub license](https://img.shields.io/github/license/cotes2020/chirpy-starter.svg?color=blue)][mit]

AI Notes is an English technical blog about agents, model APIs, and reliable AI systems. It is built from the maintained official [Chirpy Starter][starter], using the standard [**Chirpy**][chirpy] Jekyll theme and its normal GitHub Pages Actions workflow.

## Publishing

1. Add a Markdown post at `_posts/YYYY-MM-DD-slug.md`.
2. Use standard Chirpy front matter: `title`, `date`, `categories`, `tags`, and `description`.
3. Commit and push to `master`.
4. GitHub Actions runs the standard Chirpy Starter Pages workflow, builds the site, runs `htmlproofer`, and publishes the result to GitHub Pages.

The site is configured for `https://nurikk.github.io` with an empty base URL. The production build is the workflow in `.github/workflows/pages-deploy.yml`; it should remain the deployment path for this repository.

## Upstream credit

This repository adopts [cotes2020/chirpy-starter][starter] pinned at commit `beffc88713242da8bf49325674be38d171071213` (Chirpy v7.6.0). The starter and theme are distributed under the MIT License; see [LICENSE][mit].

For theme configuration and supported Markdown features, see the [Chirpy documentation][chirpy].

[gem]: https://rubygems.org/gems/jekyll-theme-chirpy
[starter]: https://github.com/cotes2020/chirpy-starter/tree/beffc88713242da8bf49325674be38d171071213
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[mit]: LICENSE
