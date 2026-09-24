# danielewiczkate.github.io

Personal site, built on [Phantom by HTML5 UP](https://html5up.net/phantom) and
served by GitHub Pages' built-in [Jekyll](https://jekyllrb.com/) build — no
local build step required, GitHub renders it on push.

## Structure

- `_config.yml` — site title/description.
- `_layouts/default.html` — the shared page chrome (header, nav, footer,
  scripts). Every page/post renders through this.
- `_posts/` — blog posts, one Markdown file per post
  (`YYYY-MM-DD-title.md`), each with a front-matter block for `title`,
  `image` (used for its home page tile) and `excerpt`.
- `index.html` — home page; loops over `site.posts` to render the tile grid,
  so a new post in `_posts/` shows up automatically.
- `elements.html` — reference page for the template's HTML components.
- `assets/`, `images/` — CSS/JS/fonts and images.

## Adding a post

Add a file to `_posts/` named `YYYY-MM-DD-slug.md`:

```markdown
---
layout: default
title: "Post Title"
image: /images/cover.jpg
excerpt: "One-line description for the home page tile."
---

Post content in Markdown.
```

## Credits

- Design: [Phantom by HTML5 UP](https://html5up.net) (CCA 3.0)
- Demo images: [Unsplash](https://unsplash.com)
- Icons: [Font Awesome](https://fontawesome.com)
- jQuery ([jquery.com](https://jquery.com)), [Responsive Tools](https://github.com/ajlkn/responsive-tools)
