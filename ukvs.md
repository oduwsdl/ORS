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
