# spark-notes

My notes from reading the Spark book


#### Spark in general
- Design principles different from MapReduce
- Does not require hdfs
- leverage lazy evaluation
- in memory computations
- the first high-level programming language for fast distributed data processing

Cluster Support:
    - Standalone Cluster Manager (included in Spark, requires Spark to be installed on each node)
    - Apache Mesos
    - Hadoop YARN

Not all transformations are 100% lazy, e.g. `sortByKey` needs to evaluate
the RDD to determine the range

Repl environment is important for debugging since errors always surface as part of
an action though it may have actually been caused by a transformation.

`.toDebugString()` useful for finding the type of RDD: Pair/Ordered/Grouped

Persist/cache forces evaluation

Misc:
    `NewHadoopRDD` - created by reading from Hadoop (presumably hdfs)


#### Cluster Resources
Can have *static* or *dynamic* allocation
dynamic - executors are added/removed as needed

by default, spark schedules jobs on a first-in, first-out basis. Though a
*fair* scheduler can be used that has a round robin approach, meaning that
a long running job will not block another job.

#### Terminology/ structure of Spark App
In brackets are the spark constructs that are split on. So each job has one
and only one action.

App --* job (action) --* Stage (wide transformation) --* task (combined narrow transformations)

**Action:** brings data out of RDD world and into some other "storage" (hdfs/console/S3)
Each action can be considered a Leaf of the DAG

Stage - set of tasks for one executor that can be completed without
communications with other executors/driver

#### DAG
Ops on RDD with known/unknown partition result in different stage boundaries
because it may require-partitioning/already-be-partitioned
When already partitioned (in the required manner) there is no need to shuffle so
it is one whole stage, whereas if the partition is unknown then the same
operations result in 2 stages.

#### misc config
`builder...getOrCreate()` ignores config set here when session already exists

#### Plugin to make dep management easier
`sbt-spark-package` e.g

    sparkVersion = ...
    sparkComponents ++= Seq("core", "sql", "hive")

#### Hive
`spark.enableHiveSupport()` (require extra jars)

There are hive specific `udf`/`udaf`s

Hive metastore - write SQL so that it is optimised to hive queries
`sc.sql("SELECT * FROM parquet.filename")`

##### tables
spark.read.table(...)
df.write.saveAsTable(...) - warning: other tools may not understand the saved format unless specific conditions are met

sqlContext.refreshTable(tableName) - use to read from the table afresh

#### Set ops
- unionAll (low cost)
- intersect (high cost)
- except (high cost)
- distinct (high cost)


#### DFs
- columnar cache format
- space efficient
- faster to encode
- don't control partitioner => can't manually avoid shuffles

#### Logical Plan
is the **Lineage of dependencies**

#### Tungsten
- specialized in-memory data structures tuned for the ops. required by Spark
- improved code gen
- specialized wire protocol*
- on heap and off heap allocations supported
- avoids memory and GC overhead of Java Objects

### reading
#### JSON
when reading (without stating schema) it samples the data to infer schema
- use `.option("samplingRatio", 0.4).load(...)` to control

tip: Do transformations on RDD to clean text then create DF from RDD
`val df = spark.read.json(rdd)`

#### DB
vendors have different JDBC implementations so we need to provide the JAR
that is required. (Not incl. in spark)

Spark includes `JdbcDialects` for DB2, Derby, MsSQL, Oracle, Postgres

#### Adding dependencies
can be done at least 2 ways:
`spark-submit --jars <path-to-jar>`
`spark-submit --packages <maven coordinates>`

- spark can run its own JDBC server

compression codec options - gzip/snappy/izo/uncompressed

- output.commuter.class is used, for S3 try ...parquet.DirectParquetOutputController

#### writing
- check file sizes (not too large or too small)

#### Datasets
- Allow writing custom Scala without UDF/UDAF required. Though at the penalty of
reduced performance. But this can be outweighed by the reduced cost to
development time!

To work with DS it is good to start with a DF, do the filtering and to
convert to a DS since DFs are better at predicate pushdown. Also be sure to
only include the min required columns in your DS - whereas in a DF, unused
columns are not read in.

- beware of large Query Plans - iterative algorithms can cause with DF/DS
(one workaround is to convert back to RDD at end of each iteration)

Datasets and spark sql can be awkward/impossible to use when the partitioning
needs to vary over the course of a job.

#### Broadcast hash join
- when rdd can fit in memory, the executors each get a full copy of the RDD
`.autoBroadcastJoinThreshold` and `.broadcastTimeout` configure when this
happens without needing to manually do it. Thus reduce/negate shuffles

- Sometimes a manual **partial** broadcast can boost performance
It is partial rather than full because the whole RDD does-not-fit-in-memory/is-large:
 so it contains just common keys OR excludes large value sets from the broadcast
- use `countByKeysApprox` to get common keys`

#### coalesce
- Is either narrow and wide transformation depending on whether inc/dec. number
of partitions.
- Can decrease level of parallelism for a stage

#### Avoid GC
- in `aggregateByKey` the seq op can be mutational on one of the accumulators
    (remember mutations are bad in other situations)
- use Arrays (instead of Tuples/objects)
- avoid Collection conversions (sometimes implicit conversions can catch you out)


#### mapPartitions
is powerful (in terms of performance and flexibility) - arbitrary functions on a partition
- take care that the function does not force *loading the entire* partition
into memory!

#### Iterator-to-Iterator transformations
- allow spark to selectively spill data to disk since they
evaluate one element at a time
- reduce GC since new objects are not created

#### Reduce Setup overhead
- mapPartitions => do setup per partition (e.g. when need util.Random()) instead of per task!
- foreachPartition similarly
- if serializable then broadcast the setup

#### Shared Variables
- broadcast var => written by driver, read by executors
- accumulators => written by executors, read by driver (take care not to mutate the obj!)

remove with `.unpersist`

e.g.

        class LazyPrng {
            @transient lazy val r = new Random()
        }

        val b = sc.broadcast(new LazyPrng())
        ... rdd.doThing( b.value.r.nextInt )


#### Accumulators
- good for process-level info e.g. time taken
- bad for data-related info e.g. counting number of invalid records
- may be evaluated multiple times - more than expected, due to re-evaluations (side-effecty)

#### persisting/checkpointing
- useful to break-up a large job that consists of a series of narrow
transformations (each task size too large since transformations condensed into one)
- when failures occur this stops needing to recompute from scratch
- reduce GC/memory strain if and only if using **off_heap** persist
- bad to persist between narrow transformations like map|filter since prevents
Catalyst from placing filter before map.
- before persisting consider if the recomputation is large relative to the cluster
and the rest of the job
- checkpointing saves partition info whereas save to storage losses partitions

##### off_heap
- allow RDD to be stored outside of executor memory
- expensive to write and read

#### Tachyon
- distributed in memory storage
- can be used as an in/out source
- off-heap
- reduced GC (since not stored as Java Objects)
- many executors share the memory pool
- data safe if an executor crashes
- best way to reuse a large RDD between spark apps

#### Unpersist
happens automatically based on LRU Last Recently Used

#### Shuffle files
Written during a shuffle - usually all of the records in each input partition
sorted by mapper
Remain for duration of app or until out of scope and GCollected
Spark can use them to avoid recomputing RDD up to shuffle
- web UI has table that show skipped stages due to shuffle files

#### Shuffle less
- preserve partitioning
- cogroup and co-located RDDs
- push computations into the shuffle

#### Shuffle better
- reduceByKey/aggregateByKey - map-side reductions
- not loading all records for a single key into memory (avoid OOM)
- even distribution of keys
- distinct keys

groupByKey results are Iterators that can't be distributed => expensive 'shuffled read'
~ has to read a lot of the shuffled data into memory
- if there are many duplicates per key

+ try to reduce the number of records first (e.g. distinct) map-side reductions
+ Iterator-to-Iterator transformations as the next operations following shuffle

#### Aggregate operations
rule:
    mem(acc') < mem(acc) + mem(v) &&
    mem(acc3) < mem(acc1) + mem(acc2)
    => is a reduction

reduceByKey/treeAggregate/aggregateByKey/foldByKey are all map-side combinators
=> combined by key **before** shuffled
=> greatly reduce shuffled read

- cogroup encounters mem errors for same reasons as groupByKey

#### Range partitioning
- problem if one range too large for executor (unbalanced data) OOM
- determines bounds by sampling so has a cost to perf compared with HashPartition
- is a transformation and an action

#### Co-located
2 RRDs are guaranteed to be co-located if partitioning was done in same
job and with same partitioner and cached

#### repartitionAndSortWithinPartitions
- pushes sort into shuffle

#### filterByRange
- better than filter when already partitioned by range - uses range information

#### sortByKey
- for compound key needs to be Tuple2 - does not support Tuple3 etc

#### SecondarySort
term from MapReduce, some sorting is done as part of the shuffle

#### Tips
Think of keys as axis for parallelization (rather than logical grouping)

