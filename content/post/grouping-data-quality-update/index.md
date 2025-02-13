---
output: hugodown::md_document
title: "Update: grouped data quality check PR merged to dbt-utils"
subtitle: ""
summary: "After a prior post on the merits of grouped data quality checks, I demo my newly merged implementation for dbt"
authors: []
tags: [data, changelog, dbt]
categories: [data, changelog, dbt]
date: 2022-08-26
lastmod: 2022-08-26
featured: false
draft: false
aliases:

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: "Photo credit to [Greyson Joralemon](https://unsplash.com/@greysonjoralemon) on Unsplash"
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: [""]
rmd_hash: 3cbf7a3ebc36534a

---

Last fall, I wrote about the [unreasonably effectiveness of grouping in data quality checks](https://www.emilyriederer.com/post/grouping-data-quality/). In this follow-up, I want to share that my [pull request](https://github.com/dbt-labs/dbt-utils/pull/633) for such features has just been merged into the development branch of the `dbt-utils` package, a common add-on to the `dbt` data transformation stack. This feature will officially "go live" in the 1.0.0 version release that is planned for later this fall.

In this brief post, I'll recall the benefits of such checks (which my original post further illustrates with NYC subway data) and demonstrate how these checks can now be implemented in `dbt-utils`.

For those interested, I'll also provide a brief overview of how I implemented this change, but I recommend checking out the PR itself for complete details.

## Recap

To recap the benefits of such checks from my initial post:

-   Some data checks can only be expressed within a group (e.g. ID values should be unique within a group but can be repeated between groups)
-   Some data checks are more precise when done by group (e.g. not only should table row-counts be equal but the counts within each group should be equal)

Of course, these benefits are more or less relevant to different types of data checks. My PR updates the following tests:

-   equal\_rowcount()
-   recency()
-   fewer\_rows\_than()
-   at\_least\_one()
-   not\_constant()
-   non\_null\_proportion()
-   sequential\_values()

Of these checks, most fall in the category of providing more rigor when being conducted at the group level. Only the `sequential_values()` test is often unable to be expressed without grouping.

## Demo

[Data tests](https://docs.getdbt.com/docs/building-a-dbt-project/tests) in `dbt` are specified in the `schema.yml` file for relevant models. Adding grouping to the tests listed above will now be as simple as adding a `group_by_columns` key-value pair to the tests, as desired, which accepts either a single variable name or a list of variables to be used for grouping.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>  - name: data_test_at_least_one
    columns:
      - name: field
        tests:
          - dbt_utils.at_least_one:
              group_by_columns: ['grouping_column']
</code></pre>

</div>

For those that have not used `dbt`'s data testing framework before, this configuration is then used to generate SQL (now with the custom `GROUP BY` clause) which are evaluated when `dbt test` is run.

## Implementation

In implementing this PR, I considered a few core principles:

-   Make this feature as unobtrusive and isolated as possible with respect to the macros broader implementation
-   Follow standard DRY principles (e.g. specifically, render needed text as few times as possible)
-   Implement consistently across macros

With these principles in mind, the majority of implementations are like that of the `recency` macro where all relevant SQL strings are pre-computed:

    {% set threshold = dbt_utils.dateadd(datepart, interval * -1, dbt_utils.current_timestamp()) %}
    {% if group_by_columns|length() > 0 %}
      {% set select_gb_cols = group_by_columns|join(' ,') + ', ' %}
      {% set groupby_gb_cols = 'group by ' + group_by_columns|join(',') %}
    {% endif %}

The main deviations to this were the sequential() macro (requiring a window function) and the equal\_rowcount()/fewer\_rows\_than() (requiring joins)

