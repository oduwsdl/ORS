# Unified Key Value Store

Unified Key Value Store (UKVS) is a textual file format for storing data that has one or more key fields for lookup and corresponding value fields.
UKVS also allows an optional object to annotate, enrich, or subdivide each record.
It uses Object Resource Stream (ORS) syntax with some well-defined fields and structure.
UKVS format has the simplicity of popular file formats such as Comma Separated Values (CSV) while being flexible like JavaScript Object Notation (JSON).
This makes it suitable for streaming and simple to process using traditional text processing tools, irrespective of the number of records in the file.
If the file is sorted, this formats enables quick lookup using binary search, even in large files, which makes it suitable for indexing records.

UKVS consists of some meta records in the header followed by the body containing data records with one line per record.
Data record entries look like the following:

```
<key fields...> <value fields...> {<optional single line JSON block>}
```

The names of the fields of the data records that belong to keys and values are advertised in the metadata headers in their respective order.
Following is a simple example (with some details omitted, which we will include later):

```
!fields {keys: ["lname", "fname"], values: ["qualification", "profession"]}
Doe   Jane    PhD     Professor             {home: "https://example.edu/members/jdoe"}
Doe   John    Masters "Business Consultant"
Roe   Richard PhD     Scientist             {dob: "May 7, 1920", awards: ["Fiction 42", "Mad Scientist"]}
Shmoe Joe     -       Assistant
```

The `!fields` metadata here describes the structure of the data records where the first two fields `lname` and `fname` are primary and secondary lookup keys respectively.
The third and fourth fields represent corresponding values as `qualification` and `profession` respectively.
These four fields are mandatory, so their missing or unknown values must be filled with a placeholder (as illustrated in the last row with a `-` sign).
Fields other than Optional Single Line JSON (OSLJ) are separated by white-spaces and multi-token values are quoted in double quotes.
While multiple consecutive white-spaces are allowed for readability, they should be avoided to save bytes as per the application needs.
So far, this format is similar to CSV/TSV files (or other tabular data formats), but the addition of the OSLJ block allows each record to have arbitrary data, unique to each record, in a structured way.
If the same information was to be represented in tabular format, it would look something like the following:

LName | FName   | Qualification | Profession          | Home                             | DoB         | Award1     | Award2
------|---------|---------------|---------------------|----------------------------------|-------------|------------|--------------
Doe   | Jane    | PhD           | Professor           | https://example.edu/members/jdoe | -           | -          | -
Doe   | John    | Masters       | Business Consultant | -                                | -           | -          | -
Roe   | Richard | PhD           | Scientist           | -                                | May 7, 1920 | Fiction 42 | Mad Scientist
Shmoe | Joe     | -             | Assistant           | -                                | -           | -          | -

This table illustrates two major issues: 1) fields with sparse values are wasteful and 2) adding a new field (e.g., `Award3`) will affect all existing rows.
Additionally, the tabular formats like CSV/TSV have no information about what fields can be used as lookup keys.
UKVS solves both of these issues as illustrated above.

This format was first realized to express holdings of a web archive (as part of the IIPC-funded Web Archive Profiling work).
While it is generic enough to be used in many key-value storage systems, in the Web Archiving community we can see numerous use cases.
Some of those use cases are described below:

* *MementoMap/Archive Profile* - MementoMap is a framework to describe the holdings of a web archive (AKA Archive Profiles) in a flexible and concise manner which is inspired by the simplicity of `sitemap` and `robots.txt`.
* *CDX File/Server* - Archival replay systems use CDX index files or servers to lookup URIs in their holdings, but the traditional format is restrictive in the sense that it does not allow arbitrary extensions and metadata for each entry that might be useful while UKVS can facilitate it.
* *Archive ACL* - Web archiving tools have no agreed upon mechanism to implement Access Control List (ACL), as a result every organization is using their own methods and additional layers to handle it, but UKVS can provide a uniform means to express ACLs both internally or publicly.
* *Archive Fixity* - Fixity or integrity of archived resources is important to establish trust in web archives, but so far very little work has been done on this, though, UKVS can provide a means to express fixity information in a flexible and future-proof manner as the specifications around this work evolve.
* *Extended TimeMap* - The `Link` format used in TimeMaps as per the RFC7089 is very limiting when it comes to express anything other than the `URI-M` and `Memento-Datetime` for `memento` relation type, but from a utility perspective many more fields can be useful such as `Status`, `Content-Digest`, `Access`, and `Damage-Score` etc. that can be enabled by UKVS.

We believe that having a unified mechanism to handle various aspects of web arching will allow us to build better and interoperable tools that will be reusable for more than one purposes.

## MementoMap/Archive Profile

MementoMap is a framework to describe the holdings of a web archive (AKA Archive Profiles) in a flexible and concise manner.
It is inspired by the simplicity of [sitemap](https://en.wikipedia.org/wiki/Sitemaps) and [robots.txt](http://www.robotstxt.org/), but different from them in some aspects:

* Unlike sitemap and robots.txt, MementoMap of an archive can be published by third parties, not just the archives.
* Unlike sitemap, MementoMap may use wildcard in URIs (similar to robots.txt) to reduce the number of records significantly.
* Unlike sitemap.txt, MementoMap can have flat pagination (generally more suitable for sorted resources) where page can link to next, previous, first, and last pages instead of a nested index of MementoMap pages.
* Unlike robots.txt, MementoMap is safe to sort, split, and merge.
* Provides means of variations in organizing various fields to optimize for space and application-specif usability while keeping the essence of records the same.
* Enables means to limit the size of the file or the number of records and dynamically adapt to it.

Starting with a simple example below:

```
!context ["http://oduwsdl.github.io/contexts/ukvs"]
!id {uri: "http://archive.example.org/"}
!fields {keys: ["surt"], values: ["frequency"]}
!meta {type: "MementoMap", name: "A Test Web Archive", year: 1996}
!meta {updated_at: "2018-09-03T13:27:52Z"}
*                   54321
com,*               10000
com,twitter)/       100
com,twitter)/*      250
uk,co,bbc)/images/* 300
```

The example above has two parts, first five lines are headers and the last five lines are data records.
The `!context` entry in the header section describes where to look for definitions and descriptions of terms used in the document.
The `!id` entry points to the web archive the MementoMap is about.
The `!fields` entry suggests that in the data records there are only two mandatory fields of which the first one is a SURT (a transformation of URIs that we describe below) of the lookup URI that is used as the key and the second field holds the frequency of archiving for a given SURT key.
The first data rwo `* 54321` suggests that there are a total of 54321 mementos of all the URIs in the archive while `com,* 10000` suggests that there are 10000 mementos that have `.com` TLD in their URIs.
Next tow entries suggest that the root page of `twitter.com` has 100 mementos, but all the URIs from `twitter.com` collectively have a total of 250 mementos.
Finally, there are 300 mementos of resources from `bbc.co.uk` who's path begins with `/images/`.

### The surt Field

Sort-friendly URI Reordering Transform (SURT), as the name suggests, is a transformation of URIs to make them sort-friendly.
In usual URIs hostname segments are delimited by dots `.` and their top level segments appear towards the right hand side while path segments are delimited using forward slashes `/` and their top-level directories are located towards the left hand side.
Due to this discrepancy, if a list of URIs is sorted alphabetically, various sub-domains of the same site might end up in different locations.
In some applications, especially in archival lookup, it is helpful to keep all records from the same site together.
To achieve this goal, hostname segments are reversed and delimited using commas so `bbc.co.uk/images` becomes `uk,co,bbc)/images`.
Before transforming to SURT, URIs are generally also canonicalized so that different variations of the same URI do not cause multiple entries in the index.
SURT are generally reversible, but in the MementoMap context (and anywhere where they are used as index keys) we can add support for partial SURTs with the help of wildcards to represent a collection of URIs.

### The frequency Field

The `frequency` field is a concise way to represent the archival activity of a URI or a set of URIs (represented by a SURT with wildcard).
In its simple form it represents the number of mementos (or URI-M counts) of a URI or all the URIs in a set.
However, it also allows to express number of unique URIs (URI-R counts) along with their URI-M counts for a given set of URIs using a wildcard as illustrated below:

```
com,apple)/*    300
com,cnn)/*      400/100
com,example)/*  200/
com,facebook)/* /50
com,google)/*   /
com,twitter)/*  0
```

The format of the frequency field here is `[<urim-count>]/[<urir-count>]`.
The first record in the example above suggests that there are 300 mementos of all the URIs from `apple.com`.
The second record suggests that there are a total of 100 unique URI-Rs (original URIs) from `cnn.com` that are archived a total of 400 times, that means on an average each URI is archived four times.
In the next entry, `200/` suggests that there are 200 URI-Ms from `example.com`, but the number of unique URI-Rs is unknown.
This entry is equivalent to saying `200` as the separator sign (i.e., forward slash `/`) is optional when only the URI-M count is known.
The decision to make the separator optional here is to save bytes in the case that is goint to be potentially more common as counting mementos (URI-Ms) and updating URI-M counts are generally easier than counting unique URI-Rs and updating the URI-R count later.
In the next entry, `/50` suggests that the number of unique URI-Rs from `facebook.com` that are archived is 50, but how many times are they archived (URI-M count) in unknown.
The separator in this case is mandatory to avoid ambiguity.
When neither URI-M nor URI-R counts are known, an explicit `/` is mandatory (as illustrated for `google.com`) to be used as the placeholder and to signify that none of the counts are known.
An explicit value of zero URI-Ms (as illustrated for `twitter.com`) gives a means to express blacklists, which can be useful for big archives like the Internet Archive, as their white-list profile might be huge, so they can instead advertise sets of URIs that they do not have (or can not serve) any mementos for, but get a lot of replay requests.
An explicit zero memento count in a MementoMap is also useful to express holdings of a bigger set, but lack of holdings in some of its subsets as illustrated below:

```
com,cnn)/*          400
com,cnn)/profiles/* 0
com,cnn)/world      0
```

This example suggests that there are a total of 400 URI-Ms for `example.com`, but zero mementos for URI-Rs that start with `cnn.com/profiles/` and zero mementos for the URI-R `cnn.com/world`.
The last two entries here are more specific subsets that override a more general record set by the first entry.

There is another case where a zero URI-M count is reported, but has a non-zero URI-R count as shown below:

```
com,yahoo)/* 0/20
```

This can be useful to report the number of URI-Rs that are blocked for legal reasons, in embargo period, or added in the seed list or frontier queue of the crawler, but not yet ready to be replayed.

Counting mementos and unique original URIs and keeping track of these numbers as the archive grows is a difficult task.
For third-party observers it is often impractical to estimate exact values of these counts.
Additionally, when counting is performed on subsets of the archival collection and merged, it is difficult to maintain the accuracy of these counts.
Luckily, in many use cases a rough estimate might be good enough while avoiding the cost of maintaining exact values.
However, it is important to acknowledge whether a frequency value is exact or a rough estimate.
This can be expressed using following symbol suffixes to either of the URI-M or URI-R counts (or both):

* No suffix for an exact value
* `+` suffix for a lower boundary
* `-` suffix for an upper boundary
* `~` for an approximate value

```
com,apple)/*    300
com,cnn)/*      400+/100
com,example)/*  200-/150~
com,facebook)/* /50+
```

The example above illustrates that there are:

* exactly 300 mementos for `apple.com` URI-Rs,
* at least 400 mementos for exactly 100 unique `cnn.com` URI-Rs,
* at most 200 mementos for approximately 150 unique `example.com` URI-Rs, and
* an unknown number of mementos for at least 50 unique `facebook.com` URI-Rs

These looseness symbols can be used (optionally) with any numeric values of URI-M and URI-R counts independently to form different combinations.

### The datetime Field

So far, we have been using `surt` as the only lookup key with no information about how the archiving activity of a URI-R or a set of URI-Rs is distributed over time.
Knowing about the temporal activity could be very helpful for Memento routing (and many other applications) for resources that were actively archived during a short period of time and hardly had any archival activity for the rest of the time.
To express the distribution of archiving activity over time the Optional Single Line JSON (OSLJ) block can be utilized as illustrated below (repeated headers are omitted for brevity):

```
!fields {keys: ["surt"], values: ["frequency"]}
com,twitter)/* 100/35 {datetime: {"2014": "20~/10", "2015": "60+/30+"}}
```

This example suggests that there are a total of exactly 100 mementos of exactly 35 unique URI-Rs from `twitter.com`.
The OSLJ block further decomposes this `frequency` value and suggests that in the year 2014 exactly 10 unique URI-Rs were archived about 20 times and more than 30 unique URI-Rs were archived more than 60 times in 2015.

This information can alternatively be expressed with a different organization as illustrated below:

```
!fields {keys: ["surt", "datetime"], values: ["frequency"]}
com,twitter)/* :    100/35
com,twitter)/* 2014 20~/10
com,twitter)/* 2015 60+/30+
```

We have moved the information buried inside of the OSLJ into a secondary key field as the first two columns `["surt", "datetime"]` are now lookup keys, making it easier to process using traditional text processing tools.
We have also avoided unnecessary bytes that were only present to format a valid JSON block and moved the name of the field in the UKVS headers instead of repeating it with each record.
However, we have introduced more lines that repeat the value of the `surt` field.
Also, if not all entries have frequency values decomposed over time, then the corresponding column will have unnecessary placeholder (a `:` symbol in this case).
So, there is clearly a trade-off here that can be evaluated based on the nature of an archive and optimized accordingly.

Potential values of the `datetime` field include valid sub-strings of the 14-digit date and time format (`YYYY[MM[DD[hh[mm[ss]]]]]`) or a range composed of such sub-strings separated by a colon `:` symbol.
If the start or the end of the range is not given it is considered the beginning of archiving and the current time respectively.

```
!fields {keys: ["surt", "datetime"], values: ["frequency"]}
com,apple)/  20160213052637 1
com,apple)/* 20160213       20
com,apple)/* 2010:2012      100
com,apple)/* :2009          50
com,apple)/* 201603:        120
com,apple)/* :              300
```

The example above (intentionally unsorted for gradual explanation) illustrates different scenarios `datetime` field can be specified.
The first entry suggests that the root page of the `apple.com` is archive exactly once at the exact time `20160213052637` (i.e., `2016/02/13 05:26:37 UTC`).
The second line suggests that URI-Rs from `apple.com` were archived a total of 20 times on February 13, 2016 (i.e., their `Memento-Datetime` in 14-digit format has a prefix of `20160213`).
The next entry of `2010:2012` suggests that there are 100 mementos of `apple.com` URIs from year 2010 to 2012.
The next line with `:2009` suggests that there are 50 mementos with `datetime` in 2009 and before.
The entry with `201603:` suggests that there are a total of 120 mementos starting from March 2016 till now.
Finally, the last line suggests that there are an over all 300 mementos of `apple.com` URI-Rs.

### Other fields

A MementoMap may also contain `frequency` information decomposed over other useful fields such as `status`, `language`, and `mime` etc.
One or more of these fields can be included in the OSLJ or as key fields (or a combination of both) as described the usage of the `datetime` field above.
Knowing the distribution of certain mementos over their `status` codes could help reduce unnecessary traffic for resources that were captured numerous times, but resulted largely in `5xx` codes.
Similarly, knowing about the number of redirects (i.e., recorded `3xx` responses) can give a better estimate of useful holdings of resources in an archive.

The OSLJ block is also open for application specific data or custom fields for specific human-friendly notes.

### Generating MementoMap

TODO: Publish scripts that generate MementoMap files and explore the possibility of integrating it into existing archival replay tools or CDX servers.

### Compacting MementoMap

TODO: Describe rules to dynamically optimize MementoMap files based on quota of resources.

### Dissemination

TODO: Describe advertisement and discovery methods of MementoMap files and their pagination options.

## CDX File/Server

TODO: Describe relevant mandatory and optional fields.

## Archive ACL

TODO: Describe relevant mandatory and optional fields.

## Archive Fixity

TODO: Describe relevant mandatory and optional fields.

## Extended TimeMap

TODO: Describe relevant mandatory and optional fields.

## UKVS Parser Considerations

TODO: Describe rules for a UKVS parser that can understand fields and express records in their expanded JSON format on demand.

## Lookup Rules

TODO: Describe how specificity and precedence rules would work when looking up for a value using one or more key fields.

## Split and Merge Rules

TODO: Describe how to transform compatible files to merge together or split
