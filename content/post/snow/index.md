---
output: hugodown::md_document
title: "How to Make R Markdown Snow"
subtitle: ""
summary: "Much like ice sculpting, applying powertools to absolutely frivolous pursuits"
authors: []
tags: [rstats, rmarkdown]
categories: [rstats, rmarkdown]
date: 2021-12-11
lastmod: 2021-12-11
featured: false
draft: false
aliases:

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: true

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: [""]
rmd_hash: 914e1908d101f4dd

---

Last year, I tweeted about how to spread holiday cheer by letting your R Markdown documents snow. After all, what better to put people in the holiday spirit than to add a random 5% probability that whatever part of a document they are trying to read will be covered?

<blockquote class="twitter-tweet">
<p lang="en" dir="ltr">
No one:<br><br>Absolutely no one:<br><br>Me: SO, I know we can\'t have a holiday party this year, but we CAN make our <a href="https://twitter.com/hashtag/rstats?src=hash&amp;ref_src=twsrc%5Etfw">\#rstats</a> R Markdown reports snow before we send them to each other <a href="https://t.co/SSBzlgb3TV">https://t.co/SSBzlgb3TV</a><br>HT to <a href="https://t.co/c7c5c5csMK">https://t.co/c7c5c5csMK</a> for the heavy lifting <a href="https://t.co/hIu7z0knR4">pic.twitter.com/hIu7z0knR4</a>
</p>
--- Emily Riederer (@EmilyRiederer) <a href="https://twitter.com/EmilyRiederer/status/1337178684868980738?ref_src=twsrc%5Etfw">December 10, 2020</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

I make no promises that this will amuse your recipients, but at least it seemed to strike a cord with other R Markdown creators. This year, I decided to write it up step-by-step. As silly as the example is, I think it demonstrates (through slight abuse) some useful features of R Markdown. Much like ice sculpting, we will apply the powertool that is R Markdown to achieve our rather fanciful end.

If you want to skip the discussed, you can check out the [full project](https://github.com/emilyriederer/demo-rmd-snow), the main [R Markdown file](https://github.com/emilyriederer/demo-rmd-snow/blob/main/index.html), or the [rendered output](https://emilyriederer.github.io/demo-rmd-snow/). The rendered output is also shown below:

![](featured.gif)

In the rest of this post, I'll touch on three R Markdown tricks and their fanciful uses:

-   **Using child documents...** to add snowflake
-   **Including raw HTML and custom CSS style...** to animate them
-   **Evaluating chunks conditionally...** to keep things seasonal

We will see how to dress up this [very important business R Markdown](https://github.com/emilyriederer/demo-rmd-snow/blob/main/index.Rmd)

Much more useful applications of these same features are discussed in the linked sections of the [R Markdown Cookbook](https://bookdown.org/yihui/rmarkdown-cookbook/).

Child documents
---------------

[Child documents](https://bookdown.org/yihui/rmarkdown-cookbook/child-document.html) allow R Markdown authors to combine multiple R Markdown files into a single final output rendered in a consistent environment. This helps create a more manageable, modular workflow if you are working on a long project or anaylsis with many distinct parts or if there are some pieces of boilerplate text or analysis that you wish to inject into many projects.

To add child documents, we create an empty R code chunk, and use the `child` chunk option to pass the path to the R Markdown file that we wish to include. In our case, we reference our `snow.Rmd` file.

    ```{r child = "snow.Rmd"}
    ```

Of course, since child documents are functionally the same as including files in the same document, we could have included this material in the same file. However, since snowflakes should clearly only be placed in *very* important documents, it is good to use best practices and take a modular approach. Tactically, this also makes it easier to "turn them on an off" at will or swap them our for New Years fireworks, Valentine's Day hearts, and more.

Including HTML and CSS
----------------------

So, what is in the [`snow.Rmd`](https://github.com/emilyriederer/demo-rmd-snow/blob/main/snow.Rmd) file?

First, we have to bring in the snowflakes themselves.

    <div class="snowflakes" aria-hidden="true">
      <div class="snowflake">
      ❅
      </div>
      <!-- many more snowflakes... -->
    </div>

Because this R Markdown will render to an HTML document, we are free to include raw HTML text the same way we include narrative, English-language text. Here, I wrap unicode snowflakes in `<divs>` so I can attach CSS classes to them.

Similarly, R Markdowns that will be rendered to HTML can use all the benefits of web technology like CSS and JavaScript. [Custom CSS](https://bookdown.org/yihui/rmarkdown-cookbook/html-css.html) can be included either with the `css` language engine or a reference in the YAML header to an external `.css` file. For compactness, I go with the former.

A `css` chunk adds CSS code used to animate the snowflake divs. This is taken nearly verbatim from [this CodePen](https://codepen.io/codeconvey/pen/xRzQay). Since this is rather lengthy, we can also use the `echo = FALSE` chunk option to not output all of the CSS in our final document.

    ```{css echo = FALSE}
    <<css goes here>>
    ```

For more tips on writing CSS for R Markown, check out my [post](https://deploy-preview-38--emilyriederer.netlify.app/post/rmarkdown-css-tips/) on finding the right selectors.

Conditional chunk evaluation
----------------------------

The above two tricks are as for as my demo goes since I only planned to render it once. However, if you are creating automated reports and fear your recipients have limited patience for animated snowflakes, we can also use R Markdown [chunk options](https://yihui.org/knitr/options/) with [variables as arguments](https://bookdown.org/yihui/rmarkdown-cookbook/chunk-variable.html) to only allow these snowflakes to appear during a certain time period.

So, for example instead of:

    ```{r child = "snow.Rmd"}
    ```

We might type:

    ```{r child = "snow.Rmd", eval = (substr(Sys.Date(), 6, 7) == 12)}
    ```

To only allow the child document to be included in December.

If we had chosen not to use child documents, we could also use chunks to achieve conditional evaluation using the [`asis` engine](https://bookdown.org/yihui/rmarkdown-cookbook/eng-asis.html).

