# LSM Tree Compactions: Leveling vs Tiering

All the pics below taken from Original Chinese version of this Article from same Author(https://zhuanlan.zhihu.com/p/112574579)

## The dilemma in LSM compaction 

The excellent performance of LSM(Log Structure Merge Tree) makes it the underlying architecture of many popular KV stores. __BigTable, Hbase, Cassandra, RocksDB__. You name it. LSM are widely used in the data product lines of various large, medium and small Internet companies, and it is one of the core architectures of many companies.

There are many introductions to the principles of LSM, so I won’t go into details in this article, I believe readers will understand. A simple overview is to write all KVs into memory (write to log at the same time for backup and recovery), and then write them to Disk SSTable sequentially and ensure that the data is sorted for quick query. As shown in below pic:

![sstable](https://github.com/zhangruiskyline/system/blob/main/images/sstable.jpg)

The advantage of LSM is that sequential writing is much more efficient than random writing, and it can often approach Disk IO speed. But there is a challenge facing the flood of writes. If we flood write to the Disk, the Disk will explode quickly, also the value of the same key often changes or new keys flood in. However, the LSM tree needs to maintain sorted data, which will result in having to search many layers to get the correct value for multiple queries. Therefore, LSM has introduced compaction, which can continuously __integrate__ the written result into sstable, and maintain the result as much as possible to update and sorted state. This is called "__Compaction__"



### Compactions illustrated

How to do compaction has actually become the core problem in the LSM tree architecture, and it is also the core balance of efficiency and cost. In order to facilitate understanding, we can assume the two extremes of compaction,  [This article](https://stratos.seas.harvard.edu/files/stratos/files/dostoevskykv.pdf) has a good explanation (by the way, I strongly recommend you to read this Lab article, very good)

![sstable](https://github.com/zhangruiskyline/system/blob/main/images/compaction_1.jpg)

-- First approach(no compactions)


The first extreme way (the upper left log scheme) is that no compaction is done at all. All data is written to the Log and then poured into the SStable. This approach, of course, has the lowest update cost and no compaction at all, just write directly to Disk. But once you want to query or update a K/V, it will cost a lot. There is no sort and no old values ​​are removed. You can only check linearly and slowly, so the efficiency is very low, and if there is no compaction, the data will be written along with it. Linear growth, faster Disk will explode.

-- Second approach(Always sort)

The second extreme way(sorted array scheme at the bottom right) is that every time a new record is written, compaction will be done immediately, and the entire data will always be updated to the latest value and sorted state, so that the query efficiency is the highest, directly according to the index, but every time The entire data need to be adjusted for every write, and sort and update are too expensive. when Write KPS becomes high, the machine cannot support such high load

To make an example, the two options are like the two extremes of cleaning up the house: the first is that you are super lazy and don't clean up at all. Things are left at random. When you throw sth new you don't care anything, but you can only find them slowly in a mess. The second is the cleanliness habit, keeping tidy up all the time, the house is completely uniform and organized, but there is no time to do other things.

Therefore, the actual DB of LSM Tree is a compromise between the two schemes. Like most of us, we may  clean up "from time to time".  LSM also needs to do compaction "from time to time". So what kind of "from time to time" is appropriate? Two basic compaction strategies will be introduced below: __level vs size/tier__


## Leveling Compaction







