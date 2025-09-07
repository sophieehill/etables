Issues with Gonçalves et al. (2025)
================
Sophie E. Hill
2025-09-06

- [Introduction](#introduction)
- [1. Estimates outside confidence
  interval](#1-estimates-outside-confidence-interval)
- [2. Frequent multiples of 0.008](#2-frequent-multiples-of-0008)
- [3. Duplicate values](#3-duplicate-values)
- [4. Estimates not centred in confidence
  interval](#4-estimates-not-centred-in-confidence-interval)
- [5. Inconsistent rounding](#5-inconsistent-rounding)
- [5. Inconsistent p-values](#5-inconsistent-p-values)
- [6. Frequent occurences of 0.000](#6-frequent-occurences-of-0000)

<style type="text/css">
.bordered {
  display: inline-block;
}
&#10;.bordered img {
  border: 1px solid black;
  display: block;
}
</style>

## Introduction

This post outlines several concerns with the quantitative results
presented in [this
paper](https://www.neurology.org/doi/10.1212/WNL.0000000000214023),
which claims to find a relationship between consumption of sweeteners
and cognitive decline:

> Gonçalves, Natalia Gomes, Euridice Martinez-Steele, Paulo A. Lotufo,
> et al. ‘Association Between Consumption of Low- and No-Calorie
> Artificial Sweeteners and Cognitive Decline’. Neurology 105, no. 7
> (2025): e214023. <https://doi.org/10.1212/WNL.0000000000214023>.

The replication materials, including the underlying data and code for
their analyses, are not available. So my comments are based only on the
results presented in the paper and the appendix.

In order to examine the results systematically, I extracted all data
from the appendix tables into a csv file, which is available in this
repo: `appendix_tables.csv`. (Note that the csv was generated with
LLM-aided extraction so there may be errors. Always check values using
the original appendix pdf.)

The appendix contains 9 tables, presenting the results of various linear
mixed models. For example:

    ## [1] "eTable 1: Association between baseline tertiles of combined low and no-caloric sweeteners and cognitive decline during eight years of follow-up"

The tables represent different specifications of the core model, where
the explanatory variable is consumption of sweeteners interacted with a
time variable and the outcome is a measure of cognitive health. Some
tables also present results on subsets of the data.

    ## Explanatory variables:

    ##  [1] "T2*Time"           "T3*Time"           "Daily consumption"
    ##  [4] "Aspartame"         "Saccharin"         "Acesulfame-k"     
    ##  [7] "Erythritol"        "Sorbitol"          "Xylitol"          
    ## [10] "Tagatose"

    ## Outcome variables:

    ## [1] "Memory"     "Verbal"     "Exec func"  "Global cog"

    ## Subsets:

    ## [1] ""       "<60"    "60+"    "w/o db" "w db"

## 1. Estimates outside confidence interval

Across the 9 tables, there are 228 estimates.

I have identified **5 cases** where the estimates lies outside its
reported confidence interval:

``` r
dat |> 
  filter(est < ci_lower | est > ci_upper) |>
  select(table_id, outcome, var, 
         subset, est, ci_lower, ci_upper)
```

    ##   table_id    outcome       var subset    est ci_lower ci_upper
    ## 1        7     Memory Aspartame    <60 -0.016   -0.001    0.001
    ## 2        9     Memory  Tagatose   w db  0.035    0.104    0.176
    ## 3        9     Verbal Aspartame w/o db  0.001   -0.002    0.000
    ## 4        9     Verbal  Tagatose   w db  0.098   -0.222    0.026
    ## 5        9 Global cog Aspartame w/o db  0.001   -0.001    0.000

The estimates in rows 2-5 all come from eTable 9 and likely represent
sign errors. This is clearest for the estimates from rows 2 and 4 since
they have larger values:

- $0.035$ $(0.104, 0.176)$ $\rightarrow$ the lower bound should be
  $-0.104$
- $0.098$ $(-0.222, 0.026)$ $\rightarrow$ the estimate should be
  $-0.098$

``` r
(-0.104 + 0.176)/2
```

    ## [1] 0.036

``` r
(-0.222 + 0.026)/2
```

    ## [1] -0.098

Rows 3 and 5 are probably also sign errors, though it’s harder to tell
with these values:

- $0.001$ $(-0.002, 0.000)$
- $0.001$ $(-0.001, 0.000)$

<div class="bordered">

<img src="images/1_table9.png" width="500" />

</div>

<br>

Row 1 is more of a mystery. This is the coefficient for Aspartame on
Memory among the \<60s in eTable 7:

$-0.016$ $(-0.001, 0.001)$

It’s not obvious to me what has gone wrong here. All the coefficients
for Aspartame on the other outcome variables in this table are 0.000.
Note the coding of the variable:

> “Apartame \[sic\], Saccharin, Acesulfame-k, Sorbitol, Xylitol, and
> Tagatose consumption ranged from 0 to 550 mg and were categorized in
> intervals of 5 mg each and analyzed as a continuous variable.
> Erythritol consumption raged from 0 to 1.8 mg and was categorized in
> intervals of 0.5 mg and analyzed as a continuous variable.”

<div class="bordered">

<img src="images/1_table7.png" width="500" />

</div>

## 2. Frequent multiples of 0.008

One odd pattern in the appendix tables is that the estimates and CI
bounds are frequently multiples of 0.008. In fact, in eTable 6, **all
estimates and CIs** are multiples of 0.008.

<div class="bordered">

<img src="images/3_table6.png" width="500" />

</div>

<br>

Across all 9 tables, over 50% (!) of estimates and confidence interval
bounds are divisible by 0.008:

``` r
cat("Proportion of values divisible by 0.008:")
```

    ## Proportion of values divisible by 0.008:

``` r
dat_8 |>
  group_by(name) |>
  summarize(prop_div = round(sum(div8)/n(), 2)) |>
  as.data.frame()
```

    ##       name prop_div
    ## 1 ci_lower     0.55
    ## 2 ci_upper     0.54
    ## 3      est     0.60

Only 11% of p-values are divisible by 0.008. This is roughly in line
with what we would expect under the null hypothesis where values are
drawn from a uniform distribution over $[0,1]$ and rounded to 3 decimal
places. (Since $1000/8 = 125$, if we drew numbers from 0 to 1,000 we
would expect $126/1001 \approx 12.5\%$ to be divisible by 8.) Going
forward I’ll exclude the p-values since they don’t seem to exhibit the
same pattern as the other values.

The frequency of 0.008 multiples varies by table, though all tables have
some. All of the estimates and CIs in eTable 6 are divisible by 0.008,
compared to only 15% in eTable 8.

``` r
cat("Proportion of values divisible by 0.008 (excluding p-values)")
```

    ## Proportion of values divisible by 0.008 (excluding p-values)

``` r
tab_props
```

    ##   table_id prop_div
    ## 1        6     1.00
    ## 2        4     0.94
    ## 3        1     0.86
    ## 4        7     0.81
    ## 5        2     0.69
    ## 6        5     0.29
    ## 7        3     0.19
    ## 8        9     0.18
    ## 9        8     0.15

It is hard to say what might be causing this unusual pattern of 0.008
multiples.

However, we can rule out a few theories:

- it is not specific to any particular version of the **explanatory
  variable**, since we see 0.008 multiples when the EV is categorical
  (tertiles of total consumption), binary (daily consumption vs
  sporadic/no consumption), or continuous (consumption of individual
  sweeteners in mg)
- it is not specific to any particular **outcome**, since we see 0.008
  multiples with all DVs (global cognition, memory, verbal fluency,
  executive function, trail-making test).
- it is not specific to any particular **subset**, since we see 0.008
  multiples in regressions on the full sample, on under/over 60s, and on
  those with/without diabetes
- it is not (solely) due to the **covariates**, since the “unadjusted”
  models in eTables 1, 2, and 3 all have 0.008 multiples

## 3. Duplicate values

Many of the tables contain identical estimates and CIs. Defining
duplicates as rows where the estimate and both bounds of the CI are
identical, within a given table, there are *46 duplicates*. Table 7 has
a particularly large number of duplicates ($19/56 = 34\%$ of all
estimates in the table have at least one duplicate).

    ## Number of duplicated est/CIs:

    ##   table_id  n
    ## 1        1  4
    ## 2        2  4
    ## 3        4  9
    ## 4        6  4
    ## 5        7 19
    ## 6        9  6

    ## Duplicated est/CIs:

    ##    id model subset          var    outcome     est ci_lower ci_upper     p
    ## 1   1 Mod 1             T2*Time     Verbal -0.0240   -0.048   -0.008 0.005
    ## 2   1 Mod 2             T2*Time     Verbal -0.0240   -0.048   -0.008 0.006
    ## 3   1 Mod 1             T3*Time Global cog -0.0240   -0.040   -0.016 0.001
    ## 4   1 Mod 2             T3*Time Global cog -0.0240   -0.040   -0.016 0.001
    ## 5   2 Mod 1             T3*Time     Verbal -0.0400   -0.056   -0.024 0.001
    ## 6   2 Mod 2             T3*Time     Verbal -0.0400   -0.056   -0.024 0.001
    ## 7   2 Mod 1             T3*Time  Exec func -0.0080   -0.032    0.008 0.395
    ## 8   2 Mod 2             T2*Time     Memory -0.0080   -0.032    0.008 0.209
    ## 9   4 Mod 1             T3*Time Global cog -0.0160   -0.048    0.008 0.145
    ## 10  4 Mod 2             T3*Time Global cog -0.0160   -0.048    0.008 0.139
    ## 11  4 Unadj             T2*Time     Memory -0.0080   -0.048    0.024 0.511
    ## 12  4 Unadj             T3*Time     Verbal -0.0080   -0.048    0.024 0.584
    ## 13  4 Mod 2             T2*Time     Memory -0.0080   -0.048    0.024 0.568
    ## 14  4 Mod 1             T2*Time  Exec func  0.0000   -0.048    0.040 0.858
    ## 15  4 Mod 1             T3*Time  Exec func  0.0000   -0.048    0.040 0.911
    ## 16  4 Mod 2             T2*Time  Exec func  0.0000   -0.048    0.040 0.889
    ## 17  4 Mod 2             T3*Time  Exec func  0.0000   -0.048    0.040 0.954
    ## 18  6  <NA>    60+      T2*Time     Memory -0.0240   -0.096    0.048 0.507
    ## 19  6  <NA>    60+      T3*Time     Memory -0.0240   -0.096    0.048 0.529
    ## 20  6  <NA>    60+      T2*Time Global cog -0.0160   -0.064    0.032 0.513
    ## 21  6  <NA>    60+      T3*Time Global cog -0.0160   -0.064    0.032 0.520
    ## 22  7  <NA>    60+ Acesulfame-k     Memory -0.0080   -0.016    0.000 0.219
    ## 23  7  <NA>    60+ Acesulfame-k  Exec func -0.0080   -0.016    0.000 0.235
    ## 24  7  <NA>    <60     Sorbitol     Memory -0.0010   -0.002    0.000 0.015
    ## 25  7  <NA>    <60     Sorbitol     Verbal -0.0010   -0.002    0.000 0.032
    ## 26  7  <NA>    60+    Aspartame     Memory  0.0000   -0.008    0.001 0.382
    ## 27  7  <NA>    60+    Aspartame  Exec func  0.0000   -0.008    0.001 0.293
    ## 28  7  <NA>    60+    Aspartame Global cog  0.0000   -0.008    0.001 0.111
    ## 29  7  <NA>    <60    Saccharin Global cog  0.0000   -0.008    0.001 0.126
    ## 30  7  <NA>    60+ Acesulfame-k Global cog  0.0000   -0.008    0.001 0.087
    ## 31  7  <NA>    <60    Saccharin     Verbal  0.0000   -0.008    0.008 0.397
    ## 32  7  <NA>    60+ Acesulfame-k     Verbal  0.0000   -0.008    0.008 0.841
    ## 33  7  <NA>    60+     Sorbitol  Exec func  0.0000   -0.002    0.001 0.613
    ## 34  7  <NA>    <60    Aspartame Global cog  0.0000   -0.002    0.001 0.214
    ## 35  7  <NA>    60+     Sorbitol     Memory  0.0000    0.000    0.000 0.814
    ## 36  7  <NA>    <60    Aspartame     Verbal  0.0000    0.000    0.000 0.366
    ## 37  7  <NA>    60+    Aspartame     Verbal  0.0000    0.000    0.000 0.831
    ## 38  7  <NA>    <60 Acesulfame-k     Verbal  0.0000    0.000    0.000 0.484
    ## 39  7  <NA>    60+     Sorbitol     Verbal  0.0000    0.000    0.000 0.573
    ## 40  7  <NA>    60+     Sorbitol Global cog  0.0000    0.000    0.000 0.611
    ## 41  9  <NA> w/o db     Sorbitol     Memory -0.0007   -0.001    0.000 0.033
    ## 42  9  <NA> w/o db     Sorbitol     Verbal -0.0007   -0.001    0.000 0.018
    ## 43  9  <NA> w/o db    Aspartame     Memory  0.0000   -0.002    0.000 0.323
    ## 44  9  <NA>   w db     Sorbitol Global cog  0.0000   -0.002    0.000 0.203
    ## 45  9  <NA> w/o db Acesulfame-k     Verbal  0.0000   -0.002    0.002 0.845
    ## 46  9  <NA>   w db     Sorbitol  Exec func  0.0000   -0.002    0.002 0.916

It is useful to see where the duplicates are within the table visually.

Here is Table 1:

<div class="bordered">

<img src="images/dupe_1.png" width="500" />

</div>

<div class="bordered">

<img src="images/dupe_2.png" width="500" />

</div>

<div class="bordered">

<img src="images/dupe_4.png" width="500" />

</div>

<div class="bordered">

<img src="images/dupe_6.png" width="500" />

</div>

<div class="bordered">

<img src="images/dupe_7.png" width="500" />

</div>

<div class="bordered">

<img src="images/dupe_9.png" width="500" />

</div>

## 4. Estimates not centred in confidence interval

## 5. Inconsistent rounding

Numerical values presented in the text of the paper and in the appendix
tables are rounded inconsistently. The majority of the values are given
to 3 decimal places, but a small number are given to 4 instead.

    ## Number of decimal places reported for estimates, CIs, and p-values:

    ##       type decimals   n
    ## 1 ci_lower        3 256
    ## 2 ci_lower        4   0
    ## 3 ci_upper        3 247
    ## 4 ci_upper        4   9
    ## 5      est        3 248
    ## 6      est        4   8
    ## 7        p        3 256
    ## 8        p        4   0

We can see the majority of the 4dp values occur in eTable 5:

    ## Rows with some values to 4dps:

    ##    table_id    outcome est_char ci_lower_char ci_upper_char     p
    ## 1         5     Memory   -0.002        -0.003       -0.0004 0.005
    ## 2         5     Memory   -0.001        -0.001       -0.0001 0.009
    ## 3         5     Verbal   -0.001        -0.002       -0.0001 0.038
    ## 4         5     Verbal   -0.002        -0.004        0.0003 0.095
    ## 5         5     Verbal  -0.0008        -0.001       -0.0003 0.002
    ## 6         5  Exec func   0.0004        -0.006         0.008 0.889
    ## 7         5  Exec func  -0.0002        -0.001        0.0004 0.443
    ## 8         5 Global cog   -0.001        -0.002       -0.0004 0.004
    ## 9         5 Global cog  -0.0006        -0.001       -0.0002 0.002
    ## 10        7 Global cog  -0.0005        -0.001       -0.0001 0.016
    ## 11        9     Memory  -0.0007        -0.001         0.000 0.033
    ## 12        9     Verbal  -0.0007        -0.001         0.000 0.018
    ## 13        9 Global cog  -0.0005        -0.001         0.000 0.011

Why were some values rounded to 4dp? It seems that these were cases
where:

- the estimate was cited directly in the text of the paper
- 4dp were necessary to remove ambiguity or confusion

For example, see this paragraph from the paper (4dp values in bold):

> Figure 3 shows the association between individual LNCS consumption and
> cognitive decline. There was a faster rate of decline in memory,
> verbal fluency, and global cognitive with higher consumption of
> aspartame (memory: β = −0.002, 95% CI −0.003 to **−0.0004**, verbal
> fluency: β = −0.001, 95% CI −0.002 to **−0.0001**, global cognition: β
> = −0.001, 95% CI −0.002 to **−0.0004**), saccharin (memory: β =
> −0.010, 95% CI −0.016 to −0.003, verbal fluency: β = −0.005, 95% CI
> −0.008 to −0.002, global cognition: β = −0.008, 95% CI −0.011 to
> −0.002), sorbitol (memory: β = −0.001, 95% CI −0.001 to **−0.0001**;
> verbal fluency: β = **−0.0008**, 95% CI −0.001 to **−0.0003**; global
> cognition: β = **−0.0006**, 95% CI −0.001 to −0.002), and xylitol
> (memory: β = −0.032, 95% CI −0.056 to −0.016; verbal fluency: β =
> −0.016, 95% CI −0.032 to −0.001; global cognition: β = −0.016, 95% CI
> −0.032 to −0.008) (Figure 3 and eTable 5).

There seem to be two distinct cases:

1.  If one bound of the confidence interval is very close to 0, then it
    is reasonable to use more decimal places to make it clear whether
    the interval covers 0. For example, in Row 1, the confidence
    interval to 3dp would be (-0.003, -0.000), which is less clear than
    (-0.003, -0.0004).

2.  If the confidence interval is very narrow and close to 0, then more
    decimal places may be needed to ensure that the estimate can be
    distinguished from the bounds. For example, in Row 13, estimate and
    CI rounded to 3dp would be -0.001 (-0.001, 0.000), which is less
    clear than -0.0005 (-0.001, 0.000).

However, the authors did not apply these rules consistently. For
example, in Row 2, the upper bound of the confidence interval has been
extended to 4dp (-0.0001) but the estimate and lower bound have not,
despite being the same value to 3dp (-0.001).

It is reasonable that the authors wanted the values cited in the text to
correspond to the same values in the appendix tables. However, their
solution to this problem was less than ideal.

It appears that the authors generated their results tables rounded to
3dp, and then manually edited specific values to 4dp. This violates
basic principles of reproducibility and introduces the risk of manual
errors or typos.

Moreover, many of the values left rounded to 3dp are essentially
uninterpretable. For example, in eTable 7 there are two estimates
reported as $0.000 (0.000; 0.000)$ with very different p-values. A much
better solution would have been to simply rescale the variables to make
the coefficients larger.

<div class="bordered">

<img src="images/4_table7.png" width="500" />

</div>

## 5. Inconsistent p-values

## 6. Frequent occurences of 0.000
