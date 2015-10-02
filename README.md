# Blog

## Jupyter (IPython) Notebooks

To convert a Jupyter (IPython) notebook to a post, but `<post-name>.ipynb` into `/_notebooks`and run

    ipython nbconvert <post-name>.ipynb --template jekyll-html

Edit the first few lines of the resulting `.html` (the yaml header) with appropriate info (probably at least the author). Tags do not need to be in `" "`, but a list of tags needs `[ ]`; for example, `[Classification, Machine Learning]` works well.

To view the post first as a draft, move <post-name>.html into `/_drafts`. 

**Note**: the name of the post will be the file name. So make stupid filenames like `SOD\ vs\ One-class\ SVM.html` and not normal filenames like `sod-vs-one-class-svm.html`.

**Note**: Spencer's hide-input-cells trick is automatically included (it's in the jekyll template and blog layouts). So ***don't*** use it in your notebook.

Once you're happy, move `<post-name>.html` to `/_posts` and rename it to `<YR>-<MO>-<DY>-<post-name>.html`. The naming convention is important for indexing.


## Installing Jekyll

The following seems to work on Ubuntu 14.04. YMMV.

    sudo apt-get install ruby-dev zlib1g-dev nodejs
    sudo gem install jekyll


## Serving the Blog

To serve the blog locally, run

    jekyll serve

Add `--host 0.0.0.0` if you serve from a VM or an internal server. The site should be available on port 4000.

Add `--drafts` if you want to see draft posts.

For development, add `--watch` if you want to automatically update the site whenever jekyll detects filechanges. This doesn't always work, though; sometimes you need to run `jekyll build` or `jekyll build --drafts` explicitly.

## Some general Jekyll stuff

 - All the HTML for generating posts and other "dynamic" content is in `/_layouts` and `/_includes`. There are also a few important HTML files in the base directory (`index.html`, `category.html`, `archive.html`, and `about.html`).

 - Jekyll will inlude any subdirectories verbatim in the generated site. (E.g. `/css` and `/img` are not jekyll specific.)

 - IPython CSS is kind of a nightmare. The standard `ipython nbconvert` inlines a bunch of CSS that they hand cut from Bootstrap, etc... (they are aware that that is a shitty thing to do, but it's what they did). That makes the generated .html files play badly with anything else. Hence the custom `jekyll-html.tpl`.

 - Jekyll reads the yaml frontmatter from any .html files in the base directory and generates pages for them (e.g. `about.html`, `category.html`, `archive.html`).

 - For more info:
    - [Jekyll Introduction](http://jekyllbootstrap.com/lessons/jekyll-introduction.html)
    - [Quick Start](http://jekyllrb.com/docs/quickstart/)


## To Do

1. Find better background images for About/Categories/Archive (Will took the current one with his phone).
2. Write some "About" text.
3. Unify the styling (btwn. Jupyter notebook posts and the rest of the blog).
4. Make more awesome posts.
5. Publish to GitHub pages.

### Credits

The blog is based on the [Jekyll version of Clean Blog Theme by Start Bootstrap](https://github.com/IronSummitMedia/startbootstrap-clean-blog-jekyll). The archive and category generating code is from [Jekyll-Bootstrap](https://github.com/plusjade/jekyll-bootstrap/).