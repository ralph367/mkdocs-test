# Features

Please refer to [Mkdocs](https://www.mkdocs.org/getting-started/) and [Mkdocs Material Theme](https://squidfunk.github.io/mkdocs-material/getting-started/) for the full documentation and features, since everything is customizable and here you can only find the most used features

## Hiding Siderbars

Add the following code to the top of your Markdown file to hide the table of content(toc) and/or the sidebar(navigation)
```
---
hide:
  - navigation
  - toc
---

# Document title
...
```

## Exclude Search

Pages can be excluded from search. Add the following lines at the top of a Markdown file:
```
---
search:
  exclude: true
---

# Document title
...
```

## Warning

!!! tip "Automatically bundle Google Fonts"

    The [built-in privacy plugin] makes it easy to use Google Fonts
    while complying with the __General Data Protection Regulation__ (GDPR),
    by automatically downloading and self-hosting the web font files.
