# dataset-parquet
Parquet metadata tools for clinical research applications, starting with CDISC

# Metadata gap
Parquet is self-descriptive, including the following column metadata
* Column Name: The name of the column.
* Column Path: The path or location of the column within nested structures.
* Data Type: The data type of the column (int32, int64, float, double, string, binary, etc.).
* Logical Type: Optional information providing semantic meaning to the data (e.g., timestamp, decimal).
* Encoding: The encoding used for the column values (e.g., PLAIN, RLE, Dictionary Encoding).
* Compression: The compression algorithm used for the column values (e.g., Snappy, Gzip).
* Repetition: How column values are repeated in the context of nested structures [REQUIRED/OPTIONAL/REPEATED]
* Statistics: Min/max values, null count, and distinct count for the column.

However compared to the traditionally SAS-based and XML-based CDISC submission standards, we miss some additional metadata such as
* dataset label
* provenance, context
* explicit link to specification / schema
* descriptive column label
* controlled terminology rule e.g. codelist name
* format (SAS-specific)
* informat (SAS-specific)

Using CDISC ODMv2 objects in a Dataset-JSON style header file plugs this gap. We use JSON here, and this could just as easily be YAML, BSON, or RDF

# CDISC ODMv2 Dataset-JSON
[Dataset JSON](https://www.cdisc.org/dataset-json) is a human-readable submission format that has many advantages over the XPT and XML exchange formats

It adds file and columnar metadata that complete the description of dataset structure and context, ideally linking to a complete specification e.g. a well-formed Define-XML document

The model used for Dataset-JSON metadata is ODMv2, which shares entities with specifications, data collection, and submission metadata.

As well as file and basic `odm:ItemDef` column metadata, data is included in Dataset-JSON as a `odm:ItemGroupData` with `.itemData` as an array of arrays or list of lists.

A disadvantage of JSON is efficiency for large datasets. This is where Parquet shines.

## CDISC Mini vs CDISC Maxi
CDISC Mini: simplified subset of the ODMv2 specification accompanies the Dataset-JSON or Dataset-Parquet. Only the basics of the `odm:ItemDef` object that describes the column, plus data about the file.

CDISC Maxi: full ODMv1 or ODMv2 dataset specification including VLM (`odm:ItemRef`, `odm:WhereClauseDef`, Biomedical Concepts) and Methods (`odm:MethodDef`) that gets submitted as Define-XML, JSON, RDF, or XLSX.

### Metadata header pointing to Parquet or vice versa
If Parquet metadata includes a reference to specification URL, parquet file can include this and be self-descriptive.

If not, a change is needed to Dataset-JSON to allow a URL to Parquet file as an alternative to the `ItemData` list of lists

# Parquet metadata
Using Arrow as a handling medium adds schema support to Parquet

PyArrow supports the following field metadata through its schema functionality:
* name
* type
* nullable
* metadata - this property can contain any JSON, allowing us to map on supporting `odm:ItemDef` content to that column

PyArrow can store file-level metadata by creating a `metadata` field and using the `metadata.metadata` property. Make sure this field doesn't get added accidentally to the resulting dataset!

### Assumptions
While Parquet supports nested groups and column-by-column encoding and compression, this project assumes flat tabular content with common encoding (UTF-8) and compression schemes across columns.

Array types might start being accepted to support R dataframes, but these can be approximated initially as JSON strings.

# Goals
Overall: provide a shared reference for CDISC parquet handling

1. Map Parquet data types and logical types to ODMv2 / Dataset-JSON types
2. Comparison and mapping of Parquet, PyArrow, Dataset-JSON metadata entities
2. Demonstrate simple Dataset-JSON support for Parquet by allowing ItemGroupData to be represented by a Parquet file (i.e. some kind of path to parquet file rather than list of lists)
3. Demonstrate Dataset-JSON roundtripping via Parquet (Dataset-Parquet) using PyArrow and its custom metadata features


# Data Types Comparison
| Data Type        | JSON                 | Dataset-JSON ItemType | PyArrow                | Parquet                |
|------------------|----------------------|-----------------------|------------------------|------------------------|
| Null             | null                 | N/A                   | pa.null()              | NA (Nullable fields)   |
| Boolean          | true/false           | boolean               | pa.bool_()             | BOOLEAN                |
| Integer          | Number               | integer               | pa.int32(), pa.int64() | INT32, INT64            |
| Float            | Number               | float                 | pa.float32(), pa.float64() | FLOAT, DOUBLE         |
| String           | String               | string                | pa.string()            | UTF8                   |
| Array            | Array                | [JSON list] string    | pa.list_()             | LIST                   |
| Object/Map       | Object               | [JSON dict] string    | pa.map_()              | MAP                    |
| Object/Struct    | Object               | [JSON dict] string    | pa.struct()            | GROUP                  |
| Timestamp        | String               | [ISO8601] string      | pa.timestamp()         | TIMESTAMP              |
| Date             | String               | [ISO8601] string      | pa.date32(), pa.date64() | DATE                  |
| Time             | String               | [ISO8601] string      | pa.time32(), pa.time64() | TIME_MILLIS, TIME_MICROS|
| Decimal          | Number               | decimal               | pa.decimal()           | DECIMAL                |
| Duration/Interval| String               | [ISO8601] string      | pa.duration()           | INTERVAL               |
| Binary           | String               | N/A                   | pa.binary()            | BYTE_ARRAY             |
| UUID             | String               | string                | pa.binary()            | FIXED_LEN_BYTE_ARRAY  |

Date/Time as a numeric is epoch-dependent, changing according to platform. To get around this, always render as a string following ISO8601 conventions.

Type and date/time handling is subject to change based on Dataset-JSON feedback.

# References
[CDISC Dataset JSON](https://www.cdisc.org/dataset-json)

[Dataset-JSON Github](https://github.com/cdisc-org/DataExchange-DatasetJson)

[Pharmaverse R-based Dataset-JSON (Atorus)](https://github.com/atorus-research/datasetjson)

[CDISC ODM](https://github.com/cdisc-org/DataExchange-ODM)

[ODMv2 Structure](https://cdisc-org.github.io/DataExchange-ODM-LinkML/)

[Labelling variables in Parquet with Arrow](https://medium.com/geekculture/label-variables-using-apache-parquet-6d03da568445)

[Adding metadata](https://medium.com/geekculture/add-metadata-to-your-dataset-using-apache-parquet-75360d2073bd)

[SAS to Parquet type conversions](https://go.documentation.sas.com/doc/en/pgmsascdc/v_035/enghdff/p1ht5enj2pwsiyn1tnxtvao89b6d.htm)

[SAS support for Dataset-JSON](https://github.com/lexjansen/dataset-json-sas)

[Get Parquet schema](https://stackoverflow.com/questions/41567081/get-schema-of-parquet-file-in-python)

[How to write file-wide metadata with PyArrow](https://stackoverflow.com/questions/52122674/how-to-write-parquet-metadata-with-pyarrow)