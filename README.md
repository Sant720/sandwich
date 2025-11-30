# Sandwich Docs

Documentation site for the Crunchy Kimchi Turkey sandwich, built with Hugo and the hugo-book theme.

## Live site

https://sandwich-webpros.vercel.app/

## Content map
- `content/docs/sandwich`: overview, core principles, ingredients, step-by-step how-to.
- `content/docs/extras`: Hugo setup log (WSL), AI disclosure, improvement ideas.
- Assets live under `static/images`.

## Prerequisites
- Hugo **extended** v0.152.2 or newer.
- Git submodules enabled (theme lives at `themes/hugo-book`).

## Setup
```
git clone https://github.com/Sant720/sandwich.git
git submodule update --init --recursive
hugo server -D
```