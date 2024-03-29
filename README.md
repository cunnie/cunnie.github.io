## blog.nono.io (cunnie.github.io)

This contains the Markdown source for my technical blog, <https://blog.nono.io>

Notes:

```bash
hugo server -D &
hugo new post/why_new_blog.md
nvim content/post/why_new_blog.md
```

Browse to <http://localhost:1313>.

When satisfied:

- change `draft` status from `true` to `false`
- commit & push
- browse to <https://blog.nono.io> to check

### References

- Overall how-to: <https://youngkin.github.io/post/createafreeblogsite/>
- Fixing first failed deploy: <https://github.com/peaceiris/actions-gh-pages#%EF%B8%8F-first-deployment-with-github_token>
- Blog theme: <https://themes.gohugo.io/themes/hugo-papermod/>
  - on GitHub: <https://github.com/adityatelange/hugo-PaperMod/wiki/Installation>
