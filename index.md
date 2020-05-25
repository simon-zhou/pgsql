## Welcome to PGSQL

PG uses a variant of L&Y algorithm for concurrent access to B-tree, which is also referred as BLink-tree. Specifically, L&Y requires a B-tree to have:
- A high-key for each page and it's greater than or equal to each tuple on that page. This high-key is only for pivot purpose when trying to find a tuple greater than any element on that page, we know that the page has been split so that we should move right.
- A **next** link to the right sibling, not only for the page split mentioned in above case, but also can be used for range scan.

In addition to that, PG also have a **prev** link to the left sibling for reverse scan.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/simon-zhou/pgsql.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
