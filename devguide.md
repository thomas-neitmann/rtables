# Developer's Guide to TableTree objects

## Read the Design Doc

Read the design document (`tabletree_design.md`) first. That will lay out definitions and basic object structure we'll need here.

## Data Values

Data values (what appears in non-empty "cells" of the table) live _only_ in `TableRow` objects. They cannot appear anywhere else.

They are stored in a `list` on the `TableRow` object and generally have no restrictions on their contents other than that the renderer is unable to


# Cheat Sheet

## Classes
See `R/00tabletrees.R` for all classes and constructor functions defined in the S4 TableTree framework.

**ALL non-virtual classes have constructor functions with identical names (including capitalization) which obfuscate the current names of the slots.**

`TableTree` - A populated (post data) Table tree node which has 0 or more `TableTree`, `ElementaryTable` or `TableRow` objecs children and a (possibly empty) content table

`ElementaryTable` - A populated Table tree node which has 0 or more children which must be `TableRow` objects

`VTableTree` - A virtual class which covers `TableTree` and `ElementaryTable`, guaranteed to have slot for children. 

`TableRow` - A populated single Row which contains a list of (possibly list/vector) values, one for each column.

`Split` - Virtual class representing a pre-data split of data defining structure/nesting in either columns or rows. See the design document for details on types of split I won't recreate them here.

`SplitVector` - a list of `Split` objects which defines a nested (sub)tree structure in either row or column space.

`InstantiatedColumnInfo` - Post-data column info which caches the column structure in tree and corresponding subset forms, as well as metadata about the columns (counts, associated extra arguments).

`PreDataColLayout`, `PreDataRowLayout` - Pre-data definitions of column/row structure 

## Accessors

See `R/tree_accessors.R` for all defined accessors within the S4 TableTree framework.

Most getters have setters with the corresponding `<-`ed version of their name. Those that don't can if we need them to in most cases. I will mention only the getters here.

`tree_children` - Get the list of children for anything modeled as a tree structure

`content_table` - Get the content table from a `TableTree` object

`obj_fmt` - a general accessor which retrieves the format associated with any supported object

`obj_label` - a general accessor which retrieves the label associated with any supported object. NOTE - some care is needed here there are currently too many concepts of label in some cases.



# Getting the Tabulation You Want

We can specify arbitrary functions in tabulation. This should allow us to get any cell contents we want provided they are they can be computed as a function of the "raw" data at tabulation time.

## Boilerplate

All examples will use the following column structure layout unless specified otherwise.

```
## we can re-use this over and over to
## generate a bunch of layouts. Pretty nice right?
collyt = NULL %>% add_colby_varlevels("ARM", "Arm") %>%
       add_colby_varlevels("SEX", "Gender")
```

## Standard Tabulation

Tabulation of data in one column is straighforward. Simply specify a function which returns either a scalar or vector to define a (single/multi valued) row or one that returns a list to define multiple rows.

If the list is named, the names are used as the row labels. if not, default row labels can also be specified in the `add_analyzed_var()` call. In the case of a non-anonymous function, subtsitute will be used to name a single row if no row names are specificed.

The variable specified is automatically selected from the subset of the data.frame corresponding to the row/column combination and passed to the tabulation function.

`fmt` takes a vector and is recycled as needed if a tabulation function generates multiple rows.

Examples:

One single valued row (mean)

```
layout = collyt %>%
       add_rowby_varlevels("RACE", "Ethnicity") %>%
       add_analyzed_var("AGE", "Age", afun = mean, fmt = "xx.xx")
```


One multi-valued row (mean and sd)

```
layout2 = collyt %>%
       add_rowby_varlevels("RACE", "Ethnicity") %>%
       add_analyzed_var("AGE", "Age", afun = function(x) c(mean = mean(x), sd = sd(x)), fmt = "xx.xx (xx.xx)")
```

Two rows (mean and median)

```
layout2 = collyt %>%
       add_rowby_varlevels("RACE", "Ethnicity") %>%
       add_analyzed_var("AGE", "Age", afun = function(x) list(mean = mean(x), median = median(x)), fmt = "xx.xx")

```

## Including NAs in data to be tabulated

By default rows with NAs in the variable being analyzed are automatically dropped *before* tabulation takes place. We can disable this by calling `add_analyzed_var()` with `inclNAs = TRUE`

Example (this is VERY silly since the mean will be NA if any NA are in the data)

```
layout = collyt %>%
       add_rowby_varlevels("RACE", "Ethnicity") %>%
       add_analyzed_var("AGE", "Age", afun = mean, fmt = "xx.xx", inclNAs = TRUE)
```

## Incorporating column or dataset totals

Tabulation functions which accept `.N_col` and/or `.N_total` arguments will be passed the column and overall observation counts, respectively.

### Examples

Proportion of observations in a column represented
by the current row. (silly but illustrates using `.N_col`)

```
layout2 = collyt %>%
       add_rowby_varlevels("RACE", "Ethnicity") %>%
       add_analyzed_var("AGE", "Age", afun = function(x, .N_col) list(prop = length(x) / .N_col), fmt = "xx.xx")
```


Columnwise proportion of total dataset (even sillier since we are ignoring the data itself, but again illustrative)

```
layout2 = collyt %>%
       add_rowby_varlevels("RACE", "Ethnicity") %>%
       add_analyzed_var("AGE", "Age", afun = function(x, .N_col, .N_total) list("column prop" = .N_col/.N_total), fmt = "xx.xx")
```

## Comparisons against "control" or "baseline" group (column)

We can declare one of the columns in our table layout the "baseline" or "control" column, by using `add_colby_varwbline()` instead of `add_colby_varlevels()`. Once this is done, we can declare comparisons against that group as part of the layout.

Setup
```
collyt2 = NULL %>% add_colby_varwbline("ARM", "Arm A", lbl = "Arm") %>%

```

### Comparing tabulation results

The standard way comparison cell values are generated is by performing a tabulation for both columns, and then passing them to the comparison function (defaults to `-` for simple differencing).

We do this by calling `add_analyzed_blinecomp`

Examples:

Differences in mean from 'baseline' column
```
layout = collyt2 %>%
       add_analyzed_blinecomp("AGE", afun = mean)
```

Ratio of means to baseline mean (note I am not saying this is a sane thing to do statistically)

```
layout = collyt2 %>%
       add_analyzed_blinecomp("AGE", afun = mean,
       compfun = function(a, b) list("ratio of means" = a/b))

```



### Comparisons based on contingency tables

For convenience we provide a helper function for comparisons which based on the 2xk table of (baseline vs column) against (levels of the variable): `add_2dtable_blinecomp()`.

When using this, we simply pass a comparsion function which accepts the table and returns what we want.

NB: We do NOT specify an analysis function here, as our comparison function does not accept 2 tabulated values, rather a single table.

```
layout = collyt2 %>%
       add_2dtable_blinecomp(var = "AGE",
       compfun = function(tab) list("1,1 value" = tab[1,1]))
```

See `tt_rsp()` in `R/tt_rsp.R` (specifically the generation of confidence intervals) for "real world" use of this convenience wrapper in practice.


### Comparisons based on full data vectors

Sometimes we may want to perform a comparison that is some complex function of both full data vectors (baseline and column).

To do this we simply use `identity` (or `function(x) x`) as our analysis function. In this case, both untabulated vectors will be passed directly to the comparison function.

NOTE: this is how the table-based comparison helper above is implemented.

Example (average ratio of the two vectors. Note unless this is pairwise data in the correct shared order this makes no sense at all statistically!!!)

```
layout = collyt2 %>%
       add_analyzed_blinecomp("AGE", afun = identity,
       compfun = function(a, b) list("mean of ratios" = mean(a/b)))


```

NB: This could occur when doing pairwise tests in certain data, though care would need to be taken to ensure identical ordering. 


## Content Row tabulation

Content rows are generally declared analogously to non-comparison data rows, except that `add_summary` or, often, the helper `add_summary_count()` instead of e.g., `add_analyzed_var()`.

The label is specified slightly differently: as a sprintf style format, where the single `%s` will be replaced with the current level at that split.

`add_summary_counts` adds observation counts and percent of column count in the form  `"xx (xx%)"`.

```
layout = collyt2 %>%
       add_rowby_varlevels("RACE", "Ethnicity", vlblvar = "ethn_lbl") %>%
       add_summary_count("RACE", lblfmt = "%s (n)")
```

The more general `add_summary()` allows us to specify a custom content function (the `cfun` argument) which can generate any desired content row(s) in the same way a tabulation function would generate data rows.

NOTE: `add_summary_counts()` is implemented as a custom content function passed to `add_summary()`.

## Column Counts

Column observation counts are not displayed by default (we could change this) but a call to `add_colcounts()` with no arguments is sufficient to add them.

Example
```
collyt3  = collyt2 %>% add_colcounts()
```


# Recursively Traversing, Subsetting and Modifying Trees

Walking trees trees, either to find or modify data or aspects of the tree, is typically a recursive operation.

Always do depth-first traversal when walking or modifying a tree (typically via recursion).  This translates to 3 rules:

1. A node should be processed before ANY of its children (e.g., process content rows)
2. A node's children should be processed before any of it's following siblings, and
3. A node's childen should be processed in order from first to last.


This is true when propogating an attribute through a tree (e.g., format) or when modifying a tree's content or row- or column- structure (both of which are represented as trees).

## A simple worked example

Here we will use recursion and S4 methods to write a `square2nd()` function which squares the value in the second column of every data and content row in a tree. Note we're squaring counts here so its a terrible idea but it illustrates the process.


The `TableTree` and `ElementaryTable` methods simply call `square2nd` again on any (content table and) children of the object and replace the old values with updated ones before returning it.

```
setGeneric("square2nd", function(obj) standardGeneric("square2nd"))
## TableTree objects (can) have content Rows
## process the content, then the children by recursive call
setMethod("square2nd", "TableTree",
	function(obj) {
    ct = content_table(obj)
    if(nrow(ct))
        content_table(obj) = square2nd(ct)
    kids = tree_children(obj)
    if(length(kids)) { 
    	newkids = lapply(kids, square2nd)
        names(newkids) = names(kids)
        tree_children(obj) = newkids
    }
    obj
})
## this will hit all Content tables as well
## as any subtrees that happen to be
## Elementary
setMethod("square2nd", "ElementaryTable",
	function(obj) {
    kids = tree_children(obj)
    if(length(kids)) {
        newkids = lapply(kids, square2nd)
        names(newkids) = names(kids)
        tree_children(obj) = newkids
    }
    obj
})
```

All of the actuaal value modification occurs in the `TableRow` method, which simply replaces the 2nd elemenent of the values list with its square and returns the modified `TableRow`

```
setMethod("square2nd", "TableRow",
	function(obj) {
    vals = row_values(obj)
    vals[[2]] = vals[[2]]^2
    row_values(obj) = vals
    obj
})
```

This "cascading methods" approach to recursion allows us to consolodate the actual modification logic within a single method we know will eventually be hit. 
The same pattern, with some additional logic, is how formats can be recursively on any node in a tree, how rows can be selected and modified more generally, and how column structure can be set or modified on an existing tree.

See, e.g., the code of `subset_cols()` and `subset_by_rownum()` for this pattern being implemented in practice.



# Getting the Rendering You Want

## Labels

## Indenting

## Formating

### Format inheritence




