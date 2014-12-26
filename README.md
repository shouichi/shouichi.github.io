# Getting Started

Install dependencies using bundler first.

```
bundle install --path vendor/bundle
```

Then you can start jekyll server and see site preview at
[http://localhost:4000](http://localhost:4000).

```
./bin/jekyll serve
```

# Writing Drafts and Publishing Them

Put a post in `_drafts` directory without timestamp. Then after writing the
post, Move the post to `_posts` with timestamp. For example, write a post as
`_drafts/example-post.md` then move it to `_posts/2014-01-01-example-post.md`
to publish.

Add `--drafts` to preview drafts.

```
./bin/jekyll serve --drafts
```

# References

- [jekyll](http://jekyllrb.com/)
