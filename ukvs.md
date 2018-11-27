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
