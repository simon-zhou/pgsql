## Welcome to PGSQL

PG uses a variant of L&Y algorithm for concurrent access to B-tree, which is also referred as BLink-tree. Specifically, L&Y requires a B-tree to have a **high-key** which is an upper bound on the keys on that page and a **next** link to the right sibling. These two additions make it possible detect a concurrent page split, which allows the tree to be searched without holding any read locks.

When a search follows a downlink to a child page, it compares the page's high key with the search key. If the search key is greater than the high key, the page must have been split concurrently, and you must follow the right-link to find the new page containing the key range you're looking for. This might need to be repeated, if the page has been split more than once.

In addition to that, PG also have a **prev** link to the left sibling for reverse scan. There are a few optimizations in PG to make B-tree more efficient:
 - HOT (Heap Only Tuple)
 - Index-only Scan
 - Suffix Truncation



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
