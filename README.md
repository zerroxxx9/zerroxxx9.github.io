# zerroxxx9.github.io

Personal website and blog powered by Jekyll with the Chirpy theme.

## Run locally

1. Enter the development environment:

```bash
nix develop
```

2. Install Ruby dependencies:

```bash
bundle install
```

3. Start the development server:

```bash
bundle exec jekyll serve
```

4. Open the site in your browser:

```text
http://127.0.0.1:4000
```

## Create new blog posts

New posts belong in the `_posts/` directory and must use this filename format:

```text
YYYY-MM-DD-title.md
```

Example:

```text
_posts/2026-05-01-my-new-post.md
```

Each post needs front matter like this:

```yaml
---
title: "My New Post"
date: 2026-05-01 10:00:00 +0200
description: "Short summary of the post."
categories: [Notes]
tags: [blog]
---
```

## Deployment

The site is prepared for GitHub Pages with Chirpy. After pushing to `main`,
GitHub Pages can build and publish it through GitHub Actions.
