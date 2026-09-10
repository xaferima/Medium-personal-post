# Xavier Riofrio — Security Blog

Personal offensive security blog covering mobile pentesting, certifications, and ethical hacking.

## Live Site

[https://xaferima.github.io/Medium-personal-post/](https://xaferima.github.io/Medium-personal-post/)

## About

This blog is a collection of technical articles on offensive security, with a focus on mobile application security. Posts include certification comparisons, pentesting methodologies, and vulnerability research.

## Tech Stack

- **Static Site Generator:** Jekyll
- **Theme:** Minima
- **Hosting:** GitHub Pages
- **Content:** Markdown

## Posts

| Date | Title | Category |
|------|-------|----------|
| 2026-09-10 | [The 9 Best Mobile Offensive Security Certifications in 2026](https://xaferima.github.io/Medium-personal-post/2026/09/10/best-mobile-offensive-security-certs-2026.html) | Certifications, Mobile |

## Local Development

```bash
# Install dependencies
bundle install

# Start local server
bundle exec jekyll serve

# Open in browser
open http://localhost:4000/Medium-personal-post/
```

## Adding a New Post

1. Create a file in `_posts/` with format `YYYY-MM-DD-<slug>.md`
2. Add Jekyll frontmatter:

```yaml
---
layout: post
title: "Post Title"
date: YYYY-MM-DD
description: "Short description for SEO"
categories: [security, mobile]
image: /Medium-personal-post/assets/images/cover.jpg
---
```

3. Write content in Markdown
4. Add cover image to `assets/images/`
5. Push to GitHub — site rebuilds automatically

## Structure

```
_posts/           ← blog posts (Markdown)
assets/images/    ← cover images
_config.yml       ← Jekyll configuration
Gemfile           ← Ruby dependencies
index.md          ← home page
about.md          ← about page
```

## Author

**Xavier Fernando Riofrio Machado**
Offensive Security Professional
