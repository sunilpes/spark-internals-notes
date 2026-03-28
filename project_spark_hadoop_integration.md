---
name: Spark Hadoop Library Integration
description: How Spark uses Hadoop libraries — FileSystem API for all storage, InputFormat/OutputFormat, Configuration, OutputCommitter, compression codecs, YARN, security/Kerberos, serialization, data locality
type: project
---

## How Spark Uses Hadoop Libraries

Spark **heavily** depends on Hadoop. It's not just for HDFS — Hadoop is used as the foundational I/O and infrastructure layer.

### 1. File System Access (the biggest one)

Spark uses Hadoop's `FileSystem` API for **ALL** storage — HDFS, S3, GCS, local FS, Azure ADLS. Spark never directly talks to these systems.

```
spark.read.parquet("s3a://bucket/data")
  → Hadoop FileSystem.get("s3a://...")     ← Hadoop resolves which FS implementation
    → S3AFileSystem (from hadoop-aws)
      → reads from S3
```

| Hadoop Class | What Spark Uses It For |
|---|---|
| `FileSystem` | Read/write/list files on any storage |
| `Path` | File path abstraction |
| `FileStatus` / `LocatedFileStatus` | File size, modification time, block locations |
| `PathFilter` | Filter files during listing |

**Key Spark files**: `SparkHadoopUtil.scala`, `HadoopFSUtils.scala`, `FileIndex.scala`

### 2. Configuration

Every Spark application carries a Hadoop `Configuration` object for storage settings, auth, codecs, etc.

```scala
// Spark forwards spark.hadoop.* configs to Hadoop
spark-submit --conf spark.hadoop.fs.s3a.access.key=XXX
  → becomes → hadoopConf.set("fs.s3a.access.key", "XXX")
```

| Hadoop Class | What Spark Uses It For |
|---|---|
| `Configuration` | All Hadoop settings (FS, compression, auth) |
| `JobConf` | Legacy MapReduce job config |

### 3. Input/Output Formats (reading & writing data)

Spark's RDD layer wraps Hadoop's `InputFormat`/`OutputFormat`:

```
sc.textFile("hdfs://...")
  → HadoopRDD → TextInputFormat (Hadoop) → RecordReader → lines

rdd.saveAsTextFile("hdfs://...")
  → SparkHadoopWriter → TextOutputFormat (Hadoop) → RecordWriter
```

| Hadoop Class | What Spark Uses It For |
|---|---|
| `InputFormat` / `RecordReader` | Reading data (splits + records) |
| `OutputFormat` / `RecordWriter` | Writing data |
| `InputSplit` / `FileSplit` | Dividing files into chunks for parallelism |
| `TextInputFormat` | `sc.textFile()` |
| `SequenceFileInputFormat` | `sc.sequenceFile()` |

**Key Spark files**: `HadoopRDD.scala`, `NewHadoopRDD.scala`, `SparkHadoopWriter.scala`

### 4. Output Committing (atomic writes)

```
DataFrame.write.parquet("/output")
  → setupJob()     ← create _temporary directory
  → commitTask()   ← each task writes to temp location
  → commitJob()    ← atomically move all files to final location
```

| Hadoop Class | What Spark Uses It For |
|---|---|
| `FileOutputCommitter` | Atomic file writes with staging |
| `OutputCommitCoordinator` | Prevents duplicate writes from speculative tasks |

### 5. Compression Codecs

```
spark.read.text("data.gz")
  → CompressionCodecFactory detects ".gz"
  → GzipCodec.createInputStream() → decompresses on read
```

| Hadoop Class | What Spark Uses It For |
|---|---|
| `CompressionCodec` | Snappy, Gzip, LZ4, Zstd |
| `CompressionCodecFactory` | Auto-detect codec from file extension |

### 6. Security & Authentication

```
spark-submit --keytab /path/to/user.keytab --principal user@REALM
  → UserGroupInformation.loginUserFromKeytab()
  → Obtain delegation tokens for HDFS, Hive, HBase
  → Distribute tokens to executors
```

| Hadoop Class | What Spark Uses It For |
|---|---|
| `UserGroupInformation` (UGI) | User identity, Kerberos login |
| `Credentials` | Token storage |
| `Token` | Delegation tokens for services |

### 7. YARN Resource Manager

```
spark-submit --master yarn
  → YarnClient.submitApplication()
  → YARN allocates containers
  → ApplicationMaster manages executor containers
```

| Hadoop Class | What Spark Uses It For |
|---|---|
| `YarnClient` | Submit app, monitor status |
| `ApplicationMaster` | Manage executor lifecycle |
| `Container` / `Resource` | CPU/memory allocation |
| `YarnConfiguration` | YARN cluster settings |

### 8. Serialization (Writable)

| Hadoop Class | What Spark Uses It For |
|---|---|
| `Writable` / `WritableComparable` | Serialize keys/values for Hadoop formats |
| `Text`, `IntWritable`, `LongWritable` | Type wrappers for Hadoop I/O |
| `NullWritable` | Placeholder for null keys |

### 9. Data Locality

```
HDFS file "users.parquet" → blocks on [node1, node2, node3]
  → Hadoop LocatedFileStatus.getBlockLocations()
  → Spark scheduler prefers node1/node2/node3 for this task
```

### Visual Summary

```
┌─────────────────────────────────────────────────────────────┐
│                        SPARK                                 │
│                                                              │
│  SQL Engine    Streaming    MLlib    GraphX    Core/RDD      │
│      │            │          │         │          │          │
│      └────────────┴──────────┴─────────┴──────────┘          │
│                           │                                  │
│              ┌────────────┴────────────┐                     │
│              ▼                         ▼                     │
│     Hadoop FileSystem API      Hadoop MapReduce API          │
│     (read/write/list files)    (InputFormat/OutputFormat)    │
│              │                         │                     │
│     Hadoop Configuration       Hadoop Serialization          │
│     (all settings)             (Writable types)              │
│              │                         │                     │
│     Hadoop Security            Hadoop Compression            │
│     (Kerberos/UGI/tokens)      (Snappy/Gzip/LZ4)           │
│              │                         │                     │
│     Hadoop YARN                Hadoop OutputCommitter        │
│     (cluster management)       (atomic writes)               │
│              │                         │                     │
└──────────────┴─────────────────────────┴─────────────────────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
           HDFS / S3 / GCS         YARN / Kerberos
           (storage)               (cluster/auth)
```

### Key Takeaway

Spark **does not** have its own file I/O, compression, or cluster management layer. It delegates all of this to Hadoop. This is why:
- Spark can read from HDFS, S3, GCS, Azure — Hadoop has FileSystem implementations for all of them
- Spark works on YARN clusters — it speaks YARN natively
- Spark can read any Hadoop-compatible data format — it wraps InputFormat
- Even on Kubernetes (non-YARN), Spark still uses Hadoop FileSystem for storage access
