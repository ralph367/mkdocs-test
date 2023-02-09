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

## Tip

```
!!! tip "Title"

    Text...
```

!!! tip "Title"

    Text...
    
    
## Warning


```
!!! warning "Title"

    Text...
```
!!! warning "Title"

    Text...
    
       
## Note

```
!!! note "Title"

    Text...
```
!!! note "Title"

    Text...
    
## Collapsible blocks

Can be used for note, tip, warning

```
??? note

    Test...
```

??? note

    Test...

