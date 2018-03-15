# Apache Parquet

> Apache Parquet is a columnar storage format available to any project
in the Hadoop ecosystem, regardless of the choice of data processing
framework, data model or programming language.

_https://parquet.apache.org/_

- open source

## History

- 2012 Twitter & Cloudera Impala both start work on prototype separately. So joined
their efforts.
- About the same time ORC was started with similar goals

## First, why store data in files rather than DB?
- performance
- simplicity e.g. portability
- economical in cost
- unstructured data

**Not optimized for:**
- querying small amounts of data
- ACID operations (Atomic Consistent Isolated Durable)

## Aims:

- Interoperability
    - Language independent

- Space efficient
    - Too much data. Storage costs.

- Query / read efficient

(Built with complex nested data structures in mind)


### Interoperability

The parquet file format **specification** is well defined at the binary level.
It is a self-describing data format, embedding the schema within the data.

- Impala implements the spec in C++

- Java Implementation

    Converter interface, enables integrating other object models:
    Other Apache formats: Avro, Thrift, Proto, and P

    - There is no intermediate object model when converting

##### Frameworks/libraries that have integrated Parquet:

- Query engines: Hive, Impala, HAWQ, IBM SQL, Drill, Tajo, Pig, Presto
- Frameworks: Spark, MapReduce, Cascading, Crunch, Scalding, Kite, NiFi
- Data Models: Avro, Thrift, ProtocolBuffers, POJO

### Space efficient

##### Columnar
- Homogenous: all values for one column in sequence
    - encode array of values rather than individual values
- [Dremel](https://blog.twitter.com/engineering/en_us/a/2013/dremel-made-simple-with-parquet.html)
used to transfer nested Data Structure into Columnar

- Best compression is often seen on sorted columns (RLE)

### Query efficient

##### Columnar
- Predicate pushdown: Read only the columns needed (IO efficient)

Optimized for the CPU, processor pipeline, to avoid jumps (make jumps
 in a way that is highly predictable by processor) and enable
parallel processing that is independent from each other.

CPU cache - keep data as close to processor as possible.
Easier to fit data inside cache with columnar format (The data in columns
that you want to ignore does not take up space)

#### Encoding

Different encodings depending on the data.

Examples:
- Country code: Dictionary
- Timestamp: Delta
- Black and white text image: Run Length Encoding

#### Supported compressions
- GZIP, SNAPPY, LZO, ZLIB

Use **Snappy** for speed of compression/decompression
Use **Gzip** for better space efficiency


## Differences/similarities to other formats

##### ORC
- Has the main aim to make Hive faster - does not care as much about Interoperability
- Also based on [Dremel paper](https://research.google.com/pubs/pub36632.html)

##### Avro
- Row based

## Benchmarks

The majority of big data analytics platform now recommend it as the most efficient,
highest performing data format. Here are recent publicly available benchmarks:

##### IBM evaluated multiple data formats for Spark SQL showed Parquet to be:

- 11 times faster than querying text files
- 75% reduced data storage thanks to built-in compression
- The only format to query large files (1 TB in size) with no errors
- Higher scan throughput on Spark 1.6


##### Cloudera examined different queries and discovered that Parquet was:

- 2 to 15 times faster than Avro, and far faster than CSV
- 72% smaller on a wide table and 25% smaller on a narrow table

##### United Airlines:

- 10 times faster than CSV on Presto and 3 times faster than CSV on Hive

##### Allstate Insurance

Compares to Avro on Spark.

[Full post here](http://blog.cloudera.com/blog/2016/04/benchmarking-apache-parquet-the-allstate-experience/)

- Spark 1.6
- Tested on 2 datasets
    - narrow:   3 columns,      82 million rows, CSV 3.9GB
    - wide:     103 columns,    694 million rows, CSV 194GB

- narrow results:
    - time to save: similar
    - row count: similar
    - GroupBy and Sum: parquet 2.6x faster
    - map: parquet 1.9x faster
    - compression: parquet 25% smaller

- wide results:
    - time to save: parquet better
    - row count: parquet 17x faster
    - GroupBy and Sum: parquet 1.8x faster
    - map: parquet 4x faster
    - compression: parquet 70% smaller
