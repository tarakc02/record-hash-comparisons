# Scope

Different things in an analysis need their own unique identifiers (e.g.
individauls, events, etc. -- anything about which you will ask "are *x* and *y*
the same?" which includes anything that needs to be counted or joined across
data sets). Individual records also need their own identifiers because

1. To track changes to records through the data processing pipeline. In the
   course of an analysis we find that record #ABC123 has an unexpected value in
   some field. Each task has an input and an output, so we can trace record
   #ABC123 back through each step to see when that value came to be -- whether
   from an original data source or through a transformation.
2. In order to maintain a database over time -- e.g. we receive batches of data
   from a partner, and as we import new batches we want to see if these are old
   records (we had them from a previous batch) or new
3. As a link to real-life. Whenever we want to review records manually, e.g. to
   apply some coding or make some other modifications, we make those changes
   via code, and the way we identify which records to modify (or remove) can be
   via an identifier.

# Concepts

- Pipeline consistency: an identifier remains the same from the start of the
  data processing pipeline through to the end. Basically any identifier you
  create that you actually write as an explicit column in the data (as opposed
  to, say, re-calculating it every time you need to use it) should fulfill
  this. The test should be how many times the identifier is calculated. If that
  happens in more than one place, it's a potential bug.

- Project consistency: This is harder -- the idea is that an identifier should
  remain consistent through the duration of the project (this could include new
  data arriving, or changes to the way data is processed, or which records are
  included in the analysis, etc.)

# Taxonomy

1. Sequential: for instance, naming the records 1-n where `n` is the number of
   records. This can fail project consistency, even if the incoming data does
   not change. Example: had a project in R where we created a sequential
   identifier after a `merge`, but `merge` has `sort = TRUE` by default, which
   sorts the output by the merge fields, and if you use `sort = FALSE`, the
   documentation says the records will be returned in "an unspecified order."
   I've used a sequential identifier in the past when reading in lines from a
   pdf that has been converted to lines of text, where the identifier looked
   like "7.14" meaning the 14th line on the 7th page.
2. Content-based: for instance a concatenation of "key" fields, or the hash of
   those fields (key fields can include every field in the data).


# Notes

- ASAP: the earlier we can create the identifier the better. Any change to code
  that is executed before the identifier is calculated could change the
  identifier of one or more records, which becomes a problem for project
  consistency.
- "Long" and "short" identifiers: Sometimes you create an identifier based on
  some key fields, e.g. name + date + event_id. I don't think of that as a
  record identifier, that is usually being done to identify some other thing in
  the data, such as a unique entity or event. It's useful to additionally have
  a "long" identifier that is distinct for every actual record in the database.


# Gotchas

## Data types

These two snippets of R code will probably generate different record
identifiers, even if they use the same hashing function:

```{r}
the_data <- read.delim("the_data.csv", sep = "|")
record_ids <- apply(the_data, 1, hash)
```

Compared to:

```{r}
the_data <- readr::read_delim("the_data.csv", delim = "|")
record_ids <- apply(the_data, 1, hash)
```

The reason is that `read.delim` has `stringsAsFactors=TRUE` by default, meaning
character fields will be read in as factor variables, and the hash of a factor
will be different from the hash of superficially the same data stored as a
regular string.

## Superficially identical records in distinct datasets

Part of our work involves integrating observations of the same events from
multiple data sources. In these cases, we want our record identifier to
distinguish between data sets also. So say we have this record in data set 1:

```
name        date          location
John Doe    1948-12-10    Palais de Chaillot, Paris
```

and then in data set 2, the same:

```
name        date          location
John Doe    1948-12-10    Palais de Chaillot, Paris
```

If we were using a content-based identifier, such as a hash id, these two
distinct records would end up with the same id! So we want to be careful, and
make sure to include the name of the dataset a record belongs to within its
identifier (or within the content fed to the hash function for a hash id).

# What's in a (column) name?

What if two objects have the same record content and data types, but different column names? Do they hash the same way?

Consider these two snippets of R code:

``` r
doe_family_1 <- tribble(
  ~NAME, ~DATE, ~LOCATION,
  "John Doe", "1948-12-10", "Paris",
  "Janet Doe", "1966-12-16", "New York City",
  "Jane Doe", "1998-07-17", "Rome") %>%
  print()
#> # A tibble: 3 x 3
#>   NAME      DATE       LOCATION     
#>   <chr>     <chr>      <chr>        
#> 1 John Doe  1948-12-10 Paris        
#> 2 Janet Doe 1966-12-16 New York City
#> 3 Jane Doe  1998-07-17 Rome
```

Compared to:

``` r
doe_family_2 <- doe_family_1 %>%
  clean_names() %>%
  print()
#> # A tibble: 3 x 3
#>   name      date       location     
#>   <chr>     <chr>      <chr>        
#> 1 John Doe  1948-12-10 Paris        
#> 2 Janet Doe 1966-12-16 New York City
#> 3 Jane Doe  1998-07-17 Rome

```

They're both tibbles with three character columns and three rows with the exact same record content. Because they have different column names, however, they hash differently.

``` r
apply(doe_family_1, 1, digest, algo = "sha1")
#> [1] "4a069856d22cbc0aa7dbe83cc33f9aa8e735740c"
#> [2] "9dfdc23516915d056feed2aa01882d6d07466fed"
#> [3] "a70ff5d42a25073a8542cf5af361d2b895963ab2"

apply(doe_family_2, 1, digest, algo = "sha1")
#> [1] "ce8deff8dc5ff4c75efae1c1882ee5e209240274"
#> [2] "9e6c274aba5b40d85f561979f82e9dc1654b8987"
#> [3] "acd02f7f003c70a5dcd118119df3aa24ed2ef70c"
```

The reason is that the column names get passed into the hash function along with the record contents when operating on dataframe rows (at least in R's implementation).

We should be thoughtful about when we generate record identifiers. By the ASAP notion, we should create this identifier before doing any sort of manipulation, even if that's just cleaning the column names, but probably after we've created a new column indicating where the data source. However, there might be compelling reasons for computing a hash id after cleaning column names. For example, a new set of data from the same source might use uppercase letters for column names instead of lowercase letters as in previous files. Cleaning the names before hashing the record contents of the new file would allow the hash id to be used to identify duplicates in previous files. 

## ?!?! stuff in the world

We had a recent experience where we were receiving data in batches from
partners, every month or two they would send an update. The updated data would
include some records they had sent us before, and some new ones, so we wanted
to be able to compare and distinguish records that we already had from a
previous batch from new records. Seems like a natural job for a hash id. Alas!
they did not format the batches consistently! In particular, the date format
would change from batch-to-batch, sometimes year first, sometimes month first,
etc.. In order to get an identifier that would be able to match records across
these different formats, we'd have to first standardize the batches. Which goes
against the earlier advice of ASAP identifier creation.

