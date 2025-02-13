---
output: hugodown::md_document
title: "Oh, I'm sure it's probably nothing"
subtitle: ""
summary: "How we do (or don't) think about null values and why the polyglot push makes it all the more important"
authors: []
tags: [rstats, python, sql, data]
categories: [rstats, python, sql, data]
date: 2022-09-05
lastmod: 2022-09-05
featured: false
draft: false
aliases:

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: "Photo credit to [Davide Ragusa](https://unsplash.com/@davideragusa) on Unsplash"
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: [""]
rmd_hash: 67ef007d16e38f9c

---

<div class="highlight">

</div>

<div class="highlight">

</div>

Language interoperability and different ways of enabling "polyglot" workflows have seemed to take centerstage in the data world recently:

-   [Apache Arrow](https://arrow.apache.org/) promises a language-independent memory format for interoperability, - [RStudio](https://www.rstudio.com/blog/rstudio-is-becoming-posit/) its rebranding as Posit to cement their place as a leader in language-agnostic data tooling,
-   RStudio simultaneously announced [Quarto](https://quarto.org/) as an interoperable alternative to RMarkdown which will treat python, Julia, and JS as first-class citizens
-   dbt has released its beta of [python models](https://docs.getdbt.com/docs/building-a-dbt-project/building-models/python-models) to extend is previously SQL-focused paradigm

As a general matter, these are all exciting advances with great potential to aid in different workflows *when used judiciously*. However, it also poses the question: what cognitive burdens do we alleviate and which do we add when our projects begin to leverage multiple languages?

Despite common data analysis tools like SQL, R, and python being high-level languages with declarative interfaces (in the case of R's `tidyverse` and python's `pandas`), successful usage still requires understanding the underlying assumptions and operations of each tool. There is not such thing as a truly declarative language; only those that generally make decisions that the user likes well-enough to ask for the "what" and delegate the "how". These differences can emerge at many different levels: such as foundational issues like whether data structures are copied or modified in-place or broader design choices like default hyperparameters in machine learning libraries (e.g. python's `scikitlearn` notoriously uses regularized logistic regression as the default for logistic regression.) Somewhere along that spectrum lies the fickle issue of handling null values.

In this post, I recap a quick case study of how incautious null handling risks data analysis validity. Then, taking a step back, I compare how R, python, and SQL behave differently when confront with null values and the implications for analysts switching between languages.

## TLDR

A summary of these different behaviors is provided below:

|                           |      **R**       |     **python**      | **SQL**  |
|:-------------------------:|:----------------:|:-------------------:|:--------:|
|   *Column Aggregation*    |        NA        | np: NA<br>pd: Value |  Value   |
| *Row-wise Transformation* |        NA        |         NA          |    NA    |
|         *Joining*         | Match by default |        Match        | No match |
|        *Filtering*        |     No match     |        Match        | No match |

## Case Study

Before comparing different languages, let's walk through a brief case study to see all the way that "lurking" nulls can surprise a junior analyst in any one language and observe a few different "contours" of the problem space.

<div class="highlight">

</div>

Consider two tables in a retailer's database. The `spend` table reports total sales by month and store identifier (null if online).

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='c'>#&gt;   STORE_ID MONTH AMT_SPEND</span></span>
<span><span class='c'>#&gt; 1        1     1  99.80583</span></span>
<span><span class='c'>#&gt; 2        2     1 100.90027</span></span>
<span><span class='c'>#&gt; 3       NA     1 100.37129</span></span>
<span><span class='c'>#&gt; 4        1     2  99.57305</span></span>
<span><span class='c'>#&gt; 5        2     2  99.02583</span></span>
<span><span class='c'>#&gt; 6       NA     2 100.81439</span></span>
<span><span class='c'>#&gt; 7        1     3  98.92842</span></span>
<span><span class='c'>#&gt; 8        2     3  99.41093</span></span>
<span><span class='c'>#&gt; 9       NA     3        NA</span></span>
<span></span></code></pre>

</div>

Similarly, the `returns` table reports returned sales at the same grain.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='c'>#&gt;   STORE_ID MONTH AMT_RETURN</span></span>
<span><span class='c'>#&gt; 1        1     1         NA</span></span>
<span><span class='c'>#&gt; 2        2     1  10.031146</span></span>
<span><span class='c'>#&gt; 3       NA     1  10.016674</span></span>
<span><span class='c'>#&gt; 4        1     2  10.032299</span></span>
<span><span class='c'>#&gt; 5        2     2  10.034934</span></span>
<span><span class='c'>#&gt; 6       NA     2   9.723547</span></span>
<span><span class='c'>#&gt; 7        1     3  10.113336</span></span>
<span><span class='c'>#&gt; 8        2     3  10.148012</span></span>
<span><span class='c'>#&gt; 9       NA     3  10.168857</span></span>
<span></span></code></pre>

</div>

In both cases, nulls are used in the `'AMT_*'` fields to denote zeros for the respective `month x store_id` combinations\`.

To calculate something as simple as the average gross spend per store across months, an analyst might attempt to write:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select 
  store_id, 
  avg(amt_spend)
from spend
group by 1
order by 1


</code></pre>

| store\_id | avg(amt\_spend) |
|----------:|----------------:|
|        NA |       100.59284 |
|         1 |        99.43577 |
|         2 |        99.77901 |

</div>

However, because SQL silently drops nulls in column aggregations, the online spend is not appropriately "penalized" for its lack of March spend. The averages across all three stores look nearly equal.

Not only is this answer "wrong", it can also be thought of as fundamentally changing the **computand** (a word I just made up. In statistics, we talk about estimands as "the conceptual thing we are trying to estimate with an estimator". Here, we aren't estimating anything -- just computing. But, there's still a concentual "thing we are trying to measure" and in this case, it's our *tools* and not our *methods* that are imposing assumptions on that) to one that answers a fundamentally different question:

Instead of measuring "average monthly spend in Q1 by store", we're measuring "averaging monthly spend in Q1 by store *conditional on* there being spend".

To obtain the correct result, one would write:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select 
  store_id, 
  -- wrong answers
  avg(amt_spend) as wrong1,  
  sum(amt_spend) / count(amt_spend) as wrong2,
  -- right answers
  sum(amt_spend) / count(1) as right1,
  avg(coalesce(amt_spend, 0)) as right2
from spend
group by 1
order by 1


</code></pre>

| store\_id |    wrong1 |    wrong2 |   right1 |   right2 |
|----------:|----------:|----------:|---------:|---------:|
|        NA | 100.59284 | 100.59284 | 67.06189 | 67.06189 |
|         1 |  99.43577 |  99.43577 | 99.43577 | 99.43577 |
|         2 |  99.77901 |  99.77901 | 99.77901 | 99.77901 |

</div>

With a better understand of gross sales, the analyst might next proceed to compute net sales.

This first requires joining the `spend` and `returns` tables. Naively, they might attempt:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select 
  spend.*,
  returns.amt_return
from 
  spend
  inner join
  returns 
  on
  spend.store_id = returns.store_id and
  spend.month = returns.month


</code></pre>

| STORE\_ID | MONTH | AMT\_SPEND | amt\_return |
|----------:|------:|-----------:|------------:|
|         1 |     1 |   99.80583 |          NA |
|         2 |     1 |  100.90027 |    10.03115 |
|         1 |     2 |   99.57305 |    10.03230 |
|         2 |     2 |   99.02583 |    10.03493 |
|         1 |     3 |   98.92842 |    10.11334 |
|         2 |     3 |   99.41093 |    10.14801 |

</div>

However, this once again fails. Why? Although SQL handled nulls "permissively" when aggregating a column, it took a stricted stance when making the comparison on `spend.store_id = returns.store_id` in the join clause. SQL doesn't recognize different nulls as equal. To the extent than null means "I dunno" versus "The field is not relevant to this observation", it's reasonable that SQL should find it hard to decide whether two "I dunno"s are equal.

Once again, this isn't a "random" or inconsequential error. Continuing to use this corrupted dataset changes the computand from "net sales by month" to "net sales by month at physical retail locations".

To remedy this, we can force `store_id` to take on a value:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select
  spend.*,
  returns.amt_return
from 
  spend
  inner join
  returns 
  on
  coalesce(spend.store_id, 999) = coalesce(returns.store_id, 999) and
  spend.month = returns.month


</code></pre>

| STORE\_ID | MONTH | AMT\_SPEND | amt\_return |
|----------:|------:|-----------:|------------:|
|         1 |     1 |   99.80583 |          NA |
|         2 |     1 |  100.90027 |   10.031146 |
|        NA |     1 |  100.37129 |   10.016674 |
|         1 |     2 |   99.57305 |   10.032299 |
|         2 |     2 |   99.02583 |   10.034934 |
|        NA |     2 |  100.81439 |    9.723547 |
|         1 |     3 |   98.92842 |   10.113336 |
|         2 |     3 |   99.41093 |   10.148012 |
|        NA |     3 |         NA |   10.168857 |

</div>

And next we proceed with computing sales by month net of returns across all stores:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select
  spend.month, 
  sum(amt_spend - amt_return) as net_spend
from 
  spend
  inner join
  returns 
  on
  coalesce(spend.store_id, 999) = coalesce(returns.store_id, 999) and
  spend.month = returns.month
group by 1
order by 1


</code></pre>

| month | net\_spend |
|------:|-----------:|
|     1 |   181.2237 |
|     2 |   269.6225 |
|     3 |   178.0780 |

</div>

However, by now, you should not be surprised that this result is also incorrect. If we inspect the sequence of computations, we realize that SQL is also stricter in its null handing in *rowwise computations* than *column-wise aggregations*. The subtraction of `amt_spend` and `amt_return` obliterates the total when either is null. So, we fail to include the gross spend at Store 1 in January simply because there were no returns (and vice versa for Internet sales in March).

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select
  spend.month, 
  spend.store_id,
  amt_spend,
  amt_return,
  amt_spend - amt_return as net_spend
from 
  spend
  inner join
  returns 
  on
  coalesce(spend.store_id, 999) = coalesce(returns.store_id, 999) and
  spend.month = returns.month


</code></pre>

| month | store\_id | amt\_spend | amt\_return | net\_spend |
|------:|----------:|-----------:|------------:|-----------:|
|     1 |         1 |   99.80583 |          NA |         NA |
|     1 |         2 |  100.90027 |   10.031146 |   90.86912 |
|     1 |        NA |  100.37129 |   10.016674 |   90.35461 |
|     2 |         1 |   99.57305 |   10.032299 |   89.54076 |
|     2 |         2 |   99.02583 |   10.034934 |   88.99089 |
|     2 |        NA |  100.81439 |    9.723547 |   91.09084 |
|     3 |         1 |   98.92842 |   10.113336 |   88.81508 |
|     3 |         2 |   99.41093 |   10.148012 |   89.26292 |
|     3 |        NA |         NA |   10.168857 |         NA |

</div>

A few ways to get the correct answer are shown below:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select
  spend.month, 
  sum(coalesce(amt_spend,0) - coalesce(amt_return,0)) as right1,
  sum(amt_spend) - sum(amt_return) as right2
from 
  spend
  inner join
  returns 
  on
  coalesce(spend.store_id, 999) = coalesce(returns.store_id, 999) and
  spend.month = returns.month
group by 1
order by 1


</code></pre>

| month |   right1 |   right2 |
|------:|---------:|---------:|
|     1 | 281.0296 | 281.0296 |
|     2 | 269.6225 | 269.6225 |
|     3 | 167.9091 | 167.9091 |

</div>

## Observations

The preceding example hopefully illustrates a few points:

-   Nulls can cause issues in the most basic of analyses
-   Beyond causing random or marginal errors, null handling changes the questions being answered
-   Even within a language, null handling may feel inconsistent (w.r.t. strictness) across different operations

So, with that, let's compare languages!

## Comparison

Below, we compare how R, SQL, and python handle column aggregation, rowwise transformation, joining, and filtering.

### Aggregation

SQL, as we saw before, simply ignores nulls in aggregation functions.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select 
  sum(x) as sum_x, 
  sum(if(x is null,1,0)) as n_null_x
from tbl


</code></pre>

| sum\_x | n\_null\_x |
|-------:|-----------:|
|      3 |          1 |

</div>

Built by and for statistician's, R is scandalized at the very idea of attempting to do math with null columns. For aggregation functions, it returns `NA` as a form of protest should any entry of the vector provided be null. (This can be overridden with the `na.rm` parameter.)

<div class='highlight'>
<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>x</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/c.html'>c</a></span><span class='o'>(</span><span class='m'>1</span>,<span class='m'>2</span>,<span class='kc'>NA</span><span class='o'>)</span></span>
<span><span class='nf'><a href='https://rdrr.io/r/base/sum.html'>sum</a></span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span></span>
<span></span>
<span><span class='nv'>df</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/data.frame.html'>data.frame</a></span><span class='o'>(</span>x <span class='o'>=</span> <span class='nv'>x</span><span class='o'>)</span></span>
<span><span class='nf'>dplyr</span><span class='nf'>::</span><span class='nf'><a href='https://dplyr.tidyverse.org/reference/summarise.html'>summarize</a></span><span class='o'>(</span><span class='nv'>df</span>, x <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/r/base/sum.html'>sum</a></span><span class='o'>(</span><span class='nv'>x</span><span class='o'>)</span><span class='o'>)</span></span>
<span><span class='c'>#&gt; [1] NA</span></span>
<span></span></code></pre>

|   x |
|----:|
|  NA |

When it comes to python, well, it depends. Base and `numpy` operations act more like R whereas `pandas` aggregation acts more like python.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>import pandas as pd
import numpy as np
x = [1,2,np.nan]
y = [3,4,5]
df = pd.DataFrame({'x':x,'y':y})
sum(x)
np.sum(x)
df.agg({'x': ['sum']})

#> nan
#> nan
#>        x
#> sum  3.0
</code></pre>

</div>

### Transformation

All of SQL, R, and python return NA when `NA`s are used in atomic or rowwise transformations.

In SQL:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select *, x-y as z
from tbl


</code></pre>

|   x |   y |   z |
|----:|----:|----:|
|   1 |   3 |  -2 |
|   2 |   4 |  -2 |
|  NA |   5 |  NA |

</div>

In R:

<div class='highlight'>
<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>df</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/data.frame.html'>data.frame</a></span><span class='o'>(</span>x <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/r/base/c.html'>c</a></span><span class='o'>(</span><span class='m'>1</span>,<span class='m'>2</span>,<span class='kc'>NA</span><span class='o'>)</span>, y <span class='o'>=</span> <span class='m'>3</span><span class='o'>:</span><span class='m'>5</span><span class='o'>)</span></span>
<span><span class='nf'>dplyr</span><span class='nf'>::</span><span class='nf'><a href='https://dplyr.tidyverse.org/reference/mutate.html'>mutate</a></span><span class='o'>(</span><span class='nv'>df</span>, z <span class='o'>=</span> <span class='nv'>x</span><span class='o'>-</span><span class='nv'>y</span><span class='o'>)</span></span>
<span></span>
<span><span class='nv'>df</span><span class='o'>$</span><span class='nv'>z</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/with.html'>with</a></span><span class='o'>(</span><span class='nv'>df</span>, <span class='nv'>x</span><span class='o'>-</span><span class='nv'>y</span><span class='o'>)</span></span>
<span><span class='nv'>df</span></span>
</code></pre>

|   x |   y |   z |
|----:|----:|----:|
|   1 |   3 |  -2 |
|   2 |   4 |  -2 |
|  NA |   5 |  NA |

|   x |   y |   z |
|----:|----:|----:|
|   1 |   3 |  -2 |
|   2 |   4 |  -2 |
|  NA |   5 |  NA |

In python:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>np.array(x) - np.array(y)
df.assign(z = lambda d: d.x - d.y)

#> array([-2., -2., nan])
#>      x  y    z
#> 0  1.0  3 -2.0
#> 1  2.0  4 -2.0
#> 2  NaN  5  NaN
</code></pre>

</div>

### Joining

The situation with joins may feel like the opposite of aggregation. Here, R and python's most popular data wrangling packages are more permissive than SQL.

As we saw in the case study, SQL does not match on nulls.

Consider `tbl1` and `tbl2` as shown below:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select * from tbl1


</code></pre>

|   A | B   | X    |
|----:|:----|:-----|
|   1 | NA  | TRUE |

</div>

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select * from tbl2


</code></pre>

|   A | B   | Y     |
|----:|:----|:------|
|   1 | NA  | FALSE |

</div>

Attempts to join return no results:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select tbl1.*, tbl2.Y 
from 
  tbl1 inner join tbl2 
  on 
  tbl1.A = tbl2.A and 
  tbl1.B = tbl2.B


</code></pre>

|   A | B   | X   | y   |
|----:|:----|:----|:----|

</div>

In contrast, default behavior for base R's `merge` and `dplyr` *does match* on nulls. (Although, either behavior can be altered with the `incomparables` or `na_matches` arguments, respectively.)

<div class='highlight'>
<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>df1</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/data.frame.html'>data.frame</a></span><span class='o'>(</span>A <span class='o'>=</span> <span class='m'>1</span>, B <span class='o'>=</span> <span class='kc'>NA</span>, X <span class='o'>=</span> <span class='kc'>TRUE</span><span class='o'>)</span></span>
<span><span class='nv'>df2</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/data.frame.html'>data.frame</a></span><span class='o'>(</span>A <span class='o'>=</span> <span class='m'>1</span>, B <span class='o'>=</span> <span class='kc'>NA</span>, Y <span class='o'>=</span> <span class='kc'>FALSE</span><span class='o'>)</span></span>
<span><span class='nf'><a href='https://rdrr.io/r/base/merge.html'>merge</a></span><span class='o'>(</span><span class='nv'>df1</span>, <span class='nv'>df2</span>, by <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/r/base/c.html'>c</a></span><span class='o'>(</span><span class='s'>"A"</span>, <span class='s'>"B"</span><span class='o'>)</span><span class='o'>)</span></span>
<span><span class='nf'>dplyr</span><span class='nf'>::</span><span class='nf'><a href='https://dplyr.tidyverse.org/reference/mutate-joins.html'>inner_join</a></span><span class='o'>(</span><span class='nv'>df1</span>, <span class='nv'>df2</span>, by <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/r/base/c.html'>c</a></span><span class='o'>(</span><span class='s'>"A"</span>, <span class='s'>"B"</span><span class='o'>)</span><span class='o'>)</span></span>
</code></pre>

|   A | B   | X    | Y     |
|----:|:----|:-----|:------|
|   1 | NA  | TRUE | FALSE |

|   A | B   | X    | Y     |
|----:|:----|:-----|:------|
|   1 | NA  | TRUE | FALSE |

Similarly, `pandas` also matches on nulls for joining.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>import numpy as np
import pandas as pd
df1 = pd.DataFrame([[1, np.nan, True]], columns = ['A','B','X'])
df2 = pd.DataFrame([[1, np.nan, False]], columns = ['A','B','Y'])
pd.merge(df1, df2, on = ['A','B'])

#>    A   B     X      Y
#> 0  1 NaN  True  False
</code></pre>

</div>

`R` and `python`'s behavior here seems most surprising. One might expect joining to work the same as raw logical evaluation works. However, neither language "likes" null comparison in its raw form. Instead, the default behavior is intentionally altered in these higher-level joining functions.

In R:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='kc'>NA</span> <span class='o'>==</span> <span class='kc'>NA</span></span>
<span><span class='c'>#&gt; [1] NA</span></span>
<span></span></code></pre>

</div>

In python:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>np.nan == np.nan

#> False
</code></pre>

</div>

### Filtering

Finally, both SQL and R drop null records used in filtering statements since comparisons with these values are incapable of returning a TRUE/FALSE value that is used to subset the rows. In python, however, pandas does preserve nulls in filter conditions.

Using the same `tbl1` shown above, we can also confirm that SQL proactively drops nulls in where clauses where they cannot be readily compared to non-null values. This seems quite consistent with its behavior in the joining case.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>select A, B, X 
from tbl1 
where B != 1


</code></pre>

|   a | b   | x   |
|----:|:----|:----|

</div>

Both base R and `dplyr` paradigms follow suit here.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>df1</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/data.frame.html'>data.frame</a></span><span class='o'>(</span>A <span class='o'>=</span> <span class='m'>1</span>, B <span class='o'>=</span> <span class='kc'>NA</span>, X <span class='o'>=</span> <span class='kc'>TRUE</span><span class='o'>)</span></span>
<span><span class='nv'>df1</span><span class='o'>[</span><span class='nv'>df1</span><span class='o'>$</span><span class='nv'>B</span> <span class='o'>!=</span> <span class='m'>1</span>,<span class='o'>]</span></span>
<span><span class='nf'>dplyr</span><span class='nf'>::</span><span class='nf'><a href='https://dplyr.tidyverse.org/reference/filter.html'>filter</a></span><span class='o'>(</span><span class='nv'>df1</span>, <span class='nv'>B</span> <span class='o'>!=</span> <span class='m'>1</span><span class='o'>)</span></span>
</code></pre>

|     |   A | B   | X   |
|:----|----:|:----|:----|
| NA  |  NA | NA  | NA  |

 A\|B \|X \|

\|--:\|:--\|:--\|

</div>

However, bucking the trend, multiple approaches to subsetting `pandas` data will not drop nulls in filtering comparisons.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'>df1 = pd.DataFrame([[1, np.nan, True]], columns = ['A','B','X'])
df1[df1.B != 1]
df1.query('B != 1')

#>    A   B     X
#> 0  1 NaN  True
#>    A   B     X
#> 0  1 NaN  True
</code></pre>

</div>

## Conclusion

In data computation and analysis, the devil is often in the details. It's not breaking news that low-level reasoning on the careful handling of null values can jeopardize the resulting analyses. However, as analysts take on increasingly complex tasks and using a plehora of different tools, it's more important than ever for both data producers and consumers to consider the choices they are making in encoding and handling these values across the stack.

