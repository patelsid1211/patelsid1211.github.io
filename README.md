# patelsid1211.github.io

Personal site of Siddharth Patel, live at https://patelsid1211.github.io

- `index.html` is the portfolio page (plain HTML, no build step).
- `blog/` lists posts. Each post is one Markdown file in `_posts/` named `YYYY-MM-DD-title.md`.
- `Siddharth_Patel.pdf` is the résumé linked from the page.

## Writing a new post

1. Create `_posts/2026-11-05-my-post-title.md`:
   ```
   ---
   layout: post
   title: "My post title"
   description: "One sentence shown in the post list."
   ---

   Post text in Markdown.
   ```
2. Commit and push. GitHub Pages builds it in about a minute.

Set `published: false` in the front matter to keep a draft off the site.
