# LSM Tree Compactions: Leveling vs Tiering

All the pics below taken from Original Chinese version of this Article from same Author(https://zhuanlan.zhihu.com/p/112574579)

## The dilemma in LSM compaction 

The excellent performance of LSM(Log Structure Merge Tree) makes it the underlying architecture of many popular KV stores. __BigTable, Hbase, Cassandra, RocksDB__. You name it. LSM are widely used in the data product lines of various large, medium and small Internet companies, and it is one of the core architectures of many companies.

 There are many introductions to the principles of LSM, so I won’t go into details in this article, I believe readers will understand. A simple overview is to write all KVs into memory (write to log at the same time for backup and recovery), and then write them to Disk SSTable sequentially and ensure that the data is sorted for quick query. As shown in below pic:

 ![sstable](https://github.com/zhangruiskyline/system/blob/master/images/sstable.jpg)

 The advantage of LSM is that sequential writing is much more efficient than random writing, and it can often approach Disk IO speed. But there is a challenge facing the flood of writes. If we flood write to the Disk, the Disk will explode quickly, also the value of the same key often changes or new keys flood in. However, the LSM tree needs to maintain sorted data, which will result in having to search many layers to get the correct value for multiple queries. Therefore, LSM has introduced compaction, which can continuously __integrate __ the written result into sstable, and maintain the result "as much as possible" to update and sorted state.



