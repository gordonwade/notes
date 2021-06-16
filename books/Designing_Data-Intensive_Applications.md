# Designing Data-Intensive Applications

## Themes
* Hiding implementation details with clean APIs
* Parallelism - CPU speed is increasing slowly, so major gains are more likely to come from parallelizing across multiple CPUs



## Chapter 2: Data Models and Query Languages
### Summary
* Many data models have been developed over time. The three main ones discussed in this chapter (in chronological order of implementation) are `hierarchical / tree`, `relational`, and `nonrelational` (with subsets of `document` and `graph`).

### Relational Model Versus Document Model

* When storing documents in a database as a string (or `JSON`, `BSON`, etc.) the `storage locality` gives an advantage (over reading from multiple tables if the data is `shredded`) if you frequently need to load most (or all) of a document for use. Conversely, if you frequently need access to small pieces of a document, this provides a disadvantage
* Some relational databases offer similar functionality with specific table features.
* Some relational databases support document access (e.g. `postgres` supports directly querying `JSON` fields).  And some document databases support joining (`MongoDB` automatically resolves document references)

### Query Languages for Data
* `SQL` is a `declarative` language (rather than `imperative`) meaning you state *what* you want, but not *how* to achieve this. The `optimizer` is responsible for the *how*.
* Declarative operations are unlikely to be broken by changes to the underlying system, where as `imperative` operations may be. This also means `declarative` operations are easier to support with parallelism.
* `MapReduce` is one model that can be used to parallelize execution of queries. Some distributed implementations of `SQL` use this, but not all.

### Graph-Like Data Models
* Graph models are best for data with heavy `many-to-many` relationships.
* Graphs consist of `nodes` and `edges`.
* `Property graphs` consist of `vertices` with:
  - Unique identifier
  - Set of incoming `edges`
  - Set of outgoing `edges`
  - Some collection of properties (`key-value` pairs)
* `Property graphs` consist of `edges` with:
  - Unique identifier
  - A `tail vertex`
  - A `head vertex`
  - A label
  - Some collection of properties
* With this model, any `vertex` can be connected to any other `vertex`.
* Any edge can be followed bi-directionally
* Labels can be used to store multiple types of information in a single graph
* All of this makes the `property graph` very flexible, as it can be used to store complex relationships and can evolve as the user needs change.
* `Cypher` is a query language specifically for property graphs. Will have to look into how relevant this is now (is everything not just `GraphQL` at this point??)
* `SQL` can do traversal similar to `Cypher` but more clumsily using `recursive common table expressions`
* `Triple-stores` offers a similar model to `property graphs` but describes the paradigm as (`subject, predicate, object`) where:
  - The `subject` is a `vertex`
  - The `object` is either a `vertex` or a value
  - The `predicate` is an `edge` connecting the two
* `Datalog` models are similar to `triple-store` but written slightly differently. It is a subset of `Prolog` which uses `rules` to define new predicates


## Chapter 3: Storage and Retrieval
### Summary
* Understanding the underlying mechanisms that different databases use for storage and retrieval can be very helpful in determining which solution makes sense for your use case.
* Understanding the different use cases (e.g. `OLTP` vs `OLAP`) is extremely important!

### Data Structures that Power Your Database
* Many databases are based on a `log` structure, using an append-only data file to store writes.
* In order to improve retrieval speed (which would be `O(n)` in a basic case), we can derive an `index` from the primary data.
* Hash-indexes are one possible solution where a map would track the location of a key in the data (potentially stored in-memory for fast lookup).
* To reduce the disk space used by our append-only log we use `compaction`, allowing us to remove duplicate keys.
  - As this reduces the size of a segment, multiple segments can then be merged
* The main downsides to a hash-map approach is the need to store it in-memory and the inability to quickly perform range queries (e.g. every key has to be looked up separately)

#### SSTables and LSM-Trees
* `Sorted String Table (SSTable)` is an option where data is stored according to key order. This improves the speed of segment merging and lookup (using a sparse index)
* Writes are problematic with this structure (because they will not be in key-order). This can be handled by maintaining writes in an in-memory key and periodically writing to disk as an `SSTable` (which can then be merged efficiently)
* The huge downside here is that we are hanging onto writes in-memory, so they would be lost in a crash. This can be avoided by maintaining a separate unsorted log file for the current "write" `SSTable` to use for restoration if a crash happens.
* A number of storage engines are built on the principle of merging and compacting sorted log files (`Log-Structured Merge-Tree`, `LSM Engines`)
* There are some specific variations to optimize for different attributes (Use of `Bloom filters` for set approximation, ordering according to `size-tier` vs. `leveled`, etc.)

#### B-Trees
* `B-trees` break the db down into `blocks` or `pages` instead of `segments`
* Each `page` contains keys interspersed with references (basically on-disk pointers) to other pages
* The number of references per page can be very high (several-hundred)
* B-trees grow by splitting pages once they become too large, so the tree always remains balanced.
  - Tree with `n` keys always has `O(log n)`
  - The number of levels is generally small (like 3 or 4) due to the branching factor allowing a huge amount of data to be stored by that point
* The B-tree algorithm requires overwriting existing data (at the original location) and sometimes updating several pages at once (i.e. when a page split occurs). This can pose a danger of data loss, which can be mitigated with a `write-ahead log (WAL)` (again used for restoring if a crash occurs)
* We also need to be aware of potential inconsistencies (e.g. read during write might get a bad result) and use `latches` to prevent this

#### B-Trees vs LSM-Trees
* B-trees are more mature (been around since 1970)
* In general, B-trees are be faster for reads, LSM-trees are faster for reads (standard disclaimer / YMMV)
*

### Transaction Processing or Analytics?
* `Transaction processing` means supporting low-latency reads and writes (not necessarily ACID) as opposed to `batch processing`.
* The access patterns for transaction usage (OLTP) vs analytics (OLAP) are very different.
* OLTP systems go for high availability and low latency, meaning running ad-hoc analytics queries against them is a bad idea!
* `Data warehousing` is one solution where read-only copies of OLTP databases can be provided for use by analysts
  - The data can be extracted periodically or via stream
  - Often times it goes through some amount of cleaning and pre-processing (ETL)
  - Data warehouses are usually relational (and can support third-party graphical query tools like Tableau)
  - Internally, they may use different database systems to optimize for different usage patterns
* Analytics schemas tend to be pretty homogeneous either in `star schema (dimensional modeling)` with a central `fact table` linking out to related `dimension tables`
  - `Snowflake schemas` are basically the same thing, but with additional normalization (e.g. `dimension tables` may be further broken down and linked to other tables)
* Both `fact` and `dimension` tables can have a huge number of columns

### Column-Oriented Storage
* For huge amounts of data, especially in systems with large numbers of columns in a given table, it can be costly to pull the entire contents of a row if you only need a few columns. `Column store` systems store the data from each column together rather than the data from each row. This relies on every column containing the record from each row in the same order.
* Column storage can work well for compression (e.g. `bitmap encoding` if the number of distinct values is much smaller than the number of rows)
* Column stores are most easily used without any sorting (rows added in the order received) but sorting based on some column value (or multiple column values) can be implemented
* If the data is being replicated to multiple machines for redundancy, we can also store each copy with a different sort order
* The optimizations that make column storage good for reading can make writing a much more intensive process. A common approach is to use the same LSM-tree in-memory storage followed by eventual merge discussed earlier
* For commonly used aggregations, relational databases might store virtual views (shortcuts for some query). For column stores specifically, `materialized views` might be used instead, which are a copy of the query results written to disk. This can cut down on processing and read time (though it adds a step to update the `materialized view` when a write is made)
* This can be done as a `data cube` or `OLAP cube` where aggregates are grouped by different dimensions (not limited to two or three dims)


## Chapter 4: Encoding and Evolution

### Summary
* When considering `evolvability`, it is important to think about the way that code and data may change over time.
* This can differ depending on the database architecture (traditional RDS have one schema enforced at all times while schema-less options allow for data in differing formats).
* Different encoding types offer different levels of support for these changes.
* Rolling upgrades allow for better evolvability (releases without downtime, etc.) but require careful though to both backward and forward compatibility

### Formats for Encoding Data
* Data is usually stored in memory while it is actively in use. When it is time to save or send this data, it has to be `encoded`, `marshalled`, or `serialized` into a format that can be written to disk or sent over the wire (e.g. JSON).
* While many languages have built in methods for encoding, they tend to have serious issues including:
  - Language lock-in
  - Need for arbitrary class instantiation (security risk!)
  - Issues with forward/backward compatibility
  - Efficiency
* Standardized encodings (JSON, XML, CSV, etc.) are one approach to this, although they have limits as well:
  - Issues with non-unicode representations and with big ints
  - The use or lack of schemas can be problematic (or time-consuming to implement)
* Binary encodings exist for these formats (in *many* variations) but it is not clear whether space savings are always worthwhile
* Some alternatives like `Thrift` and `ProtocolBuffers` take advantage of required schema to reduce the space taken up by field definitions and compact the data even further
* With the use of `field tags` updates are relatively straightforward, as the schema can change without impact to the original data. This does require the original tags to map to whatever the correct updated field is. Additionally new fields cannot be made required and old fields can never be removed
* Changes to data types can be more complicated, especially if back-compatibility is required
* `Avro` is another option that relies on a schema. It doesn't use any type of `field tag` or identifier, but rather relies on the use of compatible schemas by the reader and writer of the data (and simply uses length-prefixes and variable-length encoding to store the data)
  - `Avro` has built-in resolution for differences between the reader and writer schemas, and just relies on the app being able to provide each schema (which can be done differently depending on the use-case)
  - Even more convenient, `Avro` schemas can be dynamically generated (e.g. during a database dump) without regard to prior schema iterations
  - For statically typed languages, code generation (to implement the schema) is usually done. In dynamically typed languages, this step would be skipped.
* Many systems (RDS in particular) offer proprietary binary encodings which can be helpful when working with a proprietary driver (JDBC for example)


### Modes of Dataflow
* With database usage, new code is often written and deployed while old data may stay the same (`"data outlives code"`). Schema evolution can help this be less painful (and ensure that data doesn't all have to be rewritten)
* Another common dataflow pattern is that of requests between services (aka `service oriented architecture (SOA)` or `microservices`) in a typical client/server type interaction
* `Web services` use HTTP as an underlying protocol typically via REST or SOAP (side note, is this really still a thing?)
  - REST is a set of principles built on HTTP, emphasizing simple data formats and utilizing built-in HTTP features
  - SOAP is a standard for XML and API description is done using WSDL which is not human readable...
* `Remote procedure call (RPC)` is a precursor to APIs where function calls would be sent directly inside of network requests. This is a terrible idea due to the unreliability of network requests and the possible for information loss / lack of transparency.
  - Still, RPC systems are built on top of all of the encodings discussed earlier in this chapter. Newer RPC systems attempt to address some of the issues through the use of `futures`
* Message-passing systems are somewhere between RPC and databases in terms of their pattern
  - Requests (`messages`) are sent to a `message broker` or `message queue` which hangs onto the message until the message is delivered / dequeued
  - This is all done `asynchronously` so there is likely not an immediate response to the request
  - There is a whole section on this in the `Streaming` chapter, but the basic idea in a broker is that one process publishes to a `queue` or `topic` and the broker ensures that `consumers` or `subscribers` receive the message. Both sides (producers and consumers) can have multiple parties
* The `actor model` allows for concurrency by using individual actors with some local state. This can be distributed by integrating the message broker and actor (`distributed actor framework`)
  - `Akka`, `Orleans`, and `Erlang OTP` are all examples of this with different behavior with regard to schema changes and versioning

## Part II: Distributed Data
* The easiest way to address scaling issues is with `vertical scaling`
  - `Shared-memory architectures` allow many components to be joined together and treated as one machine
  - `Shared-disk architectures` allow data stores to be shared on networked disk between multiple machines\
* In contrast, `Shared-nothing architectures` are true `horizontal scaling` which offers much better performance at the cost of application complexity
* Data can be distributed via:
  - `Replication` providing redundancy and possible performance improvements (with distinct limits)
  - `Partitioning` which allows for greater performance gains (again with greater complexity)

## Chapter 5: Replication
### Summary
* Replication can be a good way to improve latency, availability (uptime), and/or throughput (particularly on data reads)
* The difficulty comes from handling changes to data, which can be done with `single-leader`, `multi-leader`, or `leaderless` architectures

### Leaders and Followers
* When operating a system with multiple `replicas`, every write must be processed by every replica. One way to do this is with `leader-based replication`:
  - One replica is the `leader` / `primary` which handles all write requests.
  - The `followers` or `read replicas` receive a `replication log` or `change stream` each time this happens.
  - Reads can be queried from any replica, but writes have to go to the `leader`
* This type of replication is standard on most RDS as well as some nonrelational databases and messaging solutions (e.g. `Kafka` and `RabbitMQ`)
* It is common to use a `synchronous` or `semi-synchronous` architecture here where at least (usually only) one `follower` must process the write prior to the `leader` processing the write (and returning a response to the client)
  - This ensure that there is at least one current backup at all times
  - If this specific `follower` goes down, a new `follower` would be made synchronous
* Alternatively, some setups are completely `asynchronous` which trades off durability (in the case that the leader goes down prior to writes being processed by followers) for speed
* A new `follower` can generally be spun up without reducing `leader` availability by:
  - Maintaining snapshots of the `leader` db at various timepoints (usually a built-in feature)
  - Copying that snapshot to the new node (which shouldn't require a lock)
  - Requesting all changes from the `leader` after the `log sequence number` or `binlog coordinates`
* A similar approach can be applied in the event of `follower` failure (the `follower` would keep transaction logs and request an update for all changes after the latest)
* `Leader` failure is more difficult. Resolving it involves selecting a new `leader` (possibly through consensus of existing `followers`), reconfiguring clients to direct writes to the new `leader`, and ensuring that the old `leader` acknowledges this change if it comes back online
  - In some cases multiple nodes can think they are the `leader` resulting in `split brain` which is extremely bad due to the likelihood of conflicting writes. This can be resolved by `Shoot The Other Node In The Head (STONITH)`
* Several different strategies exist for replication:
  - `Statement-based replication` involves sending the exact SQL statements to each `follower`
    - Any non-deterministic statements, auto-incrementing statements, or statements with side effects can cause huge problems. It is possible to work around these, but this replication method is falling out of favor.
  - `Write-ahead logs` can be sent instead, but due to the low-level data that they contain, they are typically incompatible with changes to the storage format (or engine). This can mean that database updates without downtime are impossible with this architecture.
  - `Logical logs` are row-based logs, decoupled from the storagfe internals. Every change to a single row is reflected as a single record. This solves some of the problems of `WAL replication`
  - `Trigger-based replication` involves setting triggers to log changes to a separate table and then reading from this table to apply updates. This is a slower method with more overhead, but can allow for greater flexibility.
*


### Problems with Replication Lag

### Multi-Leader Replication

### Leaderless Replication
