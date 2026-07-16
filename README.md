# shender.org

Source for [shender.org](https://shender.org), my personal site.  Hugo, deployed to GitHub Pages on push to `main`.  The layouts and CSS are my local layer derived from the MIT-licensed [Etch](https://github.com/LukasJoswiak/etch) theme (origins and adaptations: [`NOTICE`](NOTICE), [`LICENSES/`](LICENSES)).

## Notes to self

Hugo **Extended**, version pinned in [`.hugo-version`](.hugo-version):

```sh
hugo server -D        # dev server with drafts
hugo --gc --minify    # production build
```

Posts are `content/posts/YYYY-MM-DD-<name>.md`; the date prefix is chronology only, the URL comes from front matter:

```yaml
slug: my-post          # https://shender.org/my-post/ — never change it
tags: [emacs, ai]      # each tag gets a page + RSS feed
summary: "..."         # hand-written paragraph, served in RSS
description: "..."     # one line for search snippets and link previews
draft: true            # kept out of production
```

Math (`\(...\)`, `$$...$$`) renders at build time to HTML+MathML — no math JS.  The only JavaScript on the site is the [GoatCounter](https://www.goatcounter.com) analytics beacon (vendored `count.js`, plus a noscript pixel).

Upgrading Hugo is its own change: bump `.hugo-version` + `HUGO_VERSION`/`HUGO_SHA256` in the deploy workflow + the local binary, then check a production build and how the site looks.
