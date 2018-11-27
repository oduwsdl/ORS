# Unified Key Value Store

Unified Key Value Store (UKVS) is a textual file format for storing data that has one or more key fields for lookup and their corresponding values.
UKVS also allows an optional object per data record to annotate, enrich, or subdivide each record.
It uses Object Resource Stream (ORS) syntax with some well-defined fields and structure.
UKVS format has the simplicity of popular file formats such as Comma Separated Values (CSV) while being flexible like JavaScript Object Notation (JSON).
This makes it suitable for streaming and simple to process using traditional text processing tools, irrespective of the number of records in the file.
If the file is sorted, it enables quick lookup using binary search, even in large files, which makes it suitable for indexing records.

UKVS consists of some meta records in the header followed by data records with one line per record.
Data record entries look like the following:

```
<key fields...> <value fields...> {<optional single line JSON block>}
```

How many fields of the data records belong to keys and values and their corresponding names (in their respective order) are advertised in metadata headers.
Following is a simple example (with some details omitted, which we will include later):

```
!fields {keys: ["lname", "fname"], values: ["qualification", "profession"]}
Doe Jane PhD Professor {home: "https://example.edu/members/jdoe"}
Doe John Masters "Business Consultant"
Roe Richard PhD Scientist {dob: "May 7, 1920", awards: ["Fiction 42", "Mad Scientist"]}
Shmoe Joe - Assistant
```

The `!fields` metadata here tells about the structure of the data records where first two fields `lname` and `fname` are primary and secondary lookup keys respectively while the third and fourth fields represent corresponding values as `qualification` and `profession`.
These four fields are mandatory, so their missing or unknown values must be filled with a placeholder (as illustrated in the last row with a `-` sign).
Fields other than Optional Single Line JSON (OSLJ) are separated by white-spaces and multi-token values are quoted in double quotes.
So far, it is similar to CSV/TSV files (or other tabular data formats), but the addition of the OSLJ block allows each record to have arbitrary data, unique to each record, in a structured way.
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
* 54321
com,* 10000
com,twitter)/ 100
com,twitter)/* 250
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
com,apple)/* 300
com,cnn)/* 400/100
com,example)/* 200/
com,facebook)/* /50
com,google)/* /
com,twitter)/* 0
```

The format of the frequency field here is `[<urim-count>]/[<urir-count>]`.
The first record in the example above suggests that there are 300 mementos of all the URIs from `apple.com`.
The second record suggests that there are a total of 100 unique URI-Rs (original URIs) from `cnn.com` that are archived a total of 400 times, that means on an average each URI is archived four times.
In the next entry, `200/` suggests that there are 200 URI-Ms from `example.com`, but the number of unique URI-Rs is unknown.
This entry is equivalent to saying `200` as the separator sign (i.e., forward slash `/`) is optional when only the URI-M count is known.
The decision to make the separator optional here is to save bytes in the case that is goint to be potentially more common as counting mementos (URI-Ms) is generally easier for web archives than counting unique URI-Rs.
In the next entry, `/50` suggests that the number of unique URI-Rs from `facebook.com` that are archived is 50, but how many times are they archived (URI-M count) in unknown.
The separator in this case is mandatory to avoid ambiguity.
When neither URI-M nor URI-R counts are known, an explicit `/` is mandatory (as illustrated for `google.com`) to be used as the placeholder and to signify that none of the counts are known.
An explicit value of zero URI-Ms (as illustrated for `twitter.com`) gives a means to express blacklists, which can be useful for big archives like the Internet Archive, as their white-list profile might be huge, so they can instead advertise sets of URIs that they do not have (or can not serve) any mementos for, but get a lot of replay requests.
An explicit zero memento count in a MementoMap is also useful to express holdings of a bigger set, but lack of holdings in some of its subsets as illustrated below:

```
com,cnn)/* 400
com,cnn)/profiles/* 0
com,cnn)/world 0
```

This example suggests that there are a total of 400 URI-Ms for `example.com`, but zero mementos for URI-Rs that start with `cnn.com/profiles/` and zero mementos for the URI-R `cnn.com/world`.
The last two entries here are more specific subsets that override a more general record set by the first entry.

There is another case where a zero URI-M count is reported, but has a non-zero URI-R count as shown below:

```
com,yahoo)/* 0/20
```

This can be useful to report the number of URI-Rs that are blocked for legal reasons, in embargo period, or added in the seed list or frontier queue of the crawler, but not yet ready to be replayed.
