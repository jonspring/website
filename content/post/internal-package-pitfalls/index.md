---
output: hugodown::md_document
title: "Internal Tools Pitfalls"
subtitle: ""
summary: "What they (me) never told you about building interal tools"
authors: []
tags: [rstats, pkgdev, workflow]
categories: [rstats, pkgdev, workflow]
date: 2022-08-14
lastmod: 2022-08-14
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
rmd_hash: 2b51b8e33a558c17

---

Since releasing my first R package at Capital One in 2017, it's been no secret that I have been a proponent of internal tooling. I've espoused the benefits in numerous talks on [specific packages](/talk/tidycf/), [tactics](/talk/ent-pkg-design/), and [guiding principles](/talk/organization/).

I'm still a firm believe in the value of custom-built tools shaped for the enterprise. However, my perspective has become more nuanced and sometimes skeptical. In this post, I offer the rebuttal to my own many odes to the value of internal tools -- not to discourage their creation but to help developers better navigate pitfalls and anticipate pitfalls.

## The idealistic model

In theory, the best internal tools can create value in many different ways.

## It's not actually like open source

-   **The users are different:**
-   **The community is different:**
-   **The incentives are different:**

### It's actually too much like open source

However, many of the points above actually play into numerous *fantasies* of open-source software versys some of its realities. It's unsurprising that internally-developed tools fail to live up to the idealized version of open-source when *most open-source packages do also*.

Nadia Eghbal's excellent book [Working in Public](https://www.amazon.com/Working-Public-Making-Maintenance-Software/dp/0578675862) details the different modes that open source projects can take. Because

-   **Maintainer burnout is real:**
-   **The "bus factor" is accentuated:**
-   **Dependencies are high risk:**

### By definition, the results are not open source

## Organizational challenges

## Making it work

While I'm still a firm believe in the value of internal tooling, my perspective has become increasingly less pollyannaish over time.

what I used to think

think you get all of the benefits and none of the challenges + community, efficiency + funding!

you're probably have wrong view of open source + survivor bias + diff models (book)

it still won't be like open-source (culturally) + funding means different incentives + no positive selection + priotization (aeon) + bus problem exacerbated for long-term ownership

by definition, it's also *not* open source + no docs, stack overflow, etc. + bigger learning curve + less value additive skills

how to make it work + be ruthless about what does and doesn't make sense (drob tweet) + avoid design antipatterns + be critical

all of the problems of open source (maintainer burnout, lone wolfs, etc)

all of the problems of organizations (centralized coordination, buy-in, adoptions)
