---
name: Spark File Split and Partitioning
description: How Spark splits large files into partitions for executors — maxSplitBytes calculation, PartitionedFile splitting, bin-packing algorithm, configs, and end-to-end flow
type: project
tags: [spark-core, spark-storage]
---

## How Spark Splits Large Files for Distribution

It's a **two-phase process**: first split files into chunks, then bin-pack chunks into partitions (tasks).

---

### Phase 1: Calculate `maxSplitBytes`

**Location**: `FilePartition.scala:119-129`

```
maxSplitBytes = min(maxPartitionBytes, max(openCostInBytes, totalBytes / numCores))
```

| Config | Default | Purpose |
|--------|---------|---------|
| `spark.sql.files.maxPartitionBytes` | **128 MB** | Hard upper limit per partition |
| `spark.sql.files.openCostInBytes` | **4 MB** | Estimated cost to open a file |
| `spark.sql.files.minPartitionNum` | num cores | Minimum parallelism |

**Example**: 1 GB file, 8 cores → `bytesPerCore = 1024/8 = 128 MB` → `min(128MB, max(4MB, 128MB))` = **128 MB**

**Example**: 100 MB file, 8 cores → `bytesPerCore = 100/8 = 12.5 MB` → `min(128MB, max(4MB, 12.5MB))` = **12.5 MB** (more parallelism)

---

### Phase 2: Split Files into Chunks

**Location**: `PartitionedFileUtil.scala:27-42`

```scala
if (isSplitable) {
  // Divide file at maxSplitBytes boundaries
  (0L until file.getLen by maxSplitBytes).map { offset =>
    PartitionedFile(filePath, offset, size)
  }
} else {
  // Keep whole (e.g., gzip-compressed files)
  Seq(PartitionedFile(filePath, 0, file.getLen))
}
```

A 500 MB Parquet file with `maxSplitBytes = 128 MB` becomes:
```
PartitionedFile(file, 0, 128MB)
PartitionedFile(file, 128MB, 128MB)
PartitionedFile(file, 256MB, 128MB)
PartitionedFile(file, 384MB, 116MB)
```

**Splittable formats**: Parquet, ORC, uncompressed CSV/JSON/Text
**Non-splittable**: gzip-compressed files (must read as one chunk)

---

### Phase 3: Bin-Pack Chunks into Partitions (Tasks)

**Location**: `FilePartition.scala:58-88`

Uses **"Next Fit Decreasing"** algorithm — files sorted largest-first, greedily packed:

```scala
partitionedFiles.foreach { file =>
  if (currentSize + file.length > maxSplitBytes) {
    closePartition()  // start new partition
  }
  currentSize += file.length + openCostInBytes
  currentFiles += file
}
```

This means **small files get combined** into one partition to avoid too many tiny tasks.

---

### End-to-End Flow

```
FileSourceScanExec.createReadRDD()
  │
  ├─ 1. List files from fileIndex (with partition/filter pruning)
  ├─ 2. Calculate maxSplitBytes
  ├─ 3. Split each file into PartitionedFile chunks
  ├─ 4. Sort chunks by size (descending)
  ├─ 5. Bin-pack into FilePartition objects
  └─ 6. Create FileScanRDD(partitions) → sent to executors
```

### Concrete Example

```
Files: fileA (300MB), fileB (50MB), fileC (30MB)
Cores: 4, maxPartitionBytes: 128MB

maxSplitBytes = min(128MB, max(4MB, 380MB/4)) = min(128MB, 95MB) = 95MB

Split:
  fileA → [0-95MB], [95-190MB], [190-285MB], [285-300MB]
  fileB → [0-50MB]  (fits in one chunk)
  fileC → [0-30MB]  (fits in one chunk)

Bin-pack (sorted desc, with 4MB open cost):
  Partition 0: fileA[0-95MB]         (95MB)
  Partition 1: fileA[95-190MB]       (95MB)
  Partition 2: fileA[190-285MB]      (95MB)
  Partition 3: fileA[285-300MB] + fileB[0-50MB] + fileC[0-30MB]  (15+50+30+overhead)
```

### Special Cases

- **Bucketed tables**: Files grouped by bucket ID, bypasses bin-packing
- **HDFS locality**: Each `PartitionedFile` carries block host info so tasks prefer nodes where data lives
- **Hadoop RDD path** (older API): Delegates to `InputFormat.getSplits()` which typically splits at HDFS block boundaries (128/256 MB)

## Related Notes

- [[project_spark_hadoop_integration]] — Hadoop FileSystem API used for file listing
- [[project_spark_blockmanager_file_processing]] — When BlockManager is/isn't involved in file reads
- [[project_spark_format_aware_optimization]] — Format-aware optimizations during scan
