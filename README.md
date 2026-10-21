# tomato blog

The blog of [tomato](https://github.com/tomatool/tomato), published at
**https://tomatool.github.io/blog/**.

A [Hugo](https://gohugo.io) site with its own theme, [`themes/tomato`](themes/tomato),
in the palette of `tomato ui`: warm near-black, the red fruit and the green stem,
and the Gherkin phase colours for syntax. Dark by default; light follows the
reader's system or the toggle.

## Writing a post

```bash
hugo new content posts/my-post.md   # draft = true until you flip it
hugo server -D                      # http://localhost:1313/blog/
```

Every push to `main` builds the site and publishes it to GitHub Pages
([Publish](.github/workflows/pages.yml)).
