<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [LSM Tree Compactions: Leveling vs Tiering](#lsm-tree-compactions-leveling-vs-tiering)
  - [The dilemma in LSM compaction](#the-dilemma-in-lsm-compaction)
    - [Compactions illustrated](#compactions-illustrated)
      - [First approach(no compactions)](#first-approachno-compactions)
      - [Second approach(Always sort)](#second-approachalways-sort)
  - [Leveling Compaction](#leveling-compaction)
  - [Size/Tier Compaction](#sizetier-compaction)
  - [Leveling vs Tier Compaction](#leveling-vs-tier-compaction)
    - [Amplification](#amplification)
      - [Write amplification](#write-amplification)
      - [Read amplification](#read-amplification)
      - [Size amplification](#size-amplification)
  - [Last but not least](#last-but-not-least)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# LSM Tree Compactions: Leveling vs Tiering

All the pics below taken from Original Chinese version of this Article from same Author(https://zhuanlan.zhihu.com/p/112574579)

## The dilemma in LSM compaction 

The excellent performance of LSM(Log Structure Merge Tree) makes it the underlying architecture of many popular KV stores. __BigTable, Hbase, Cassandra, RocksDB__. You name it. LSM are widely used in the data product lines of various large, medium and small Internet companies, and it is one of the core architectures of many companies.

There are many introductions to the principles of LSM, so I won’t go into details in this article, I believe readers will understand. A simple overview is to write all KVs into memory (write to log at the same time for backup and recovery), and then write them to Disk SSTable sequentially and ensure that the data is sorted for quick query. As shown in below pic:

![sstable](https://github.com/zhangruiskyline/system/blob/main/images/sstable.jpg)

The advantage of LSM is that sequential writing is much more efficient than random writing, and it can often approach Disk IO speed. But there is a challenge facing the flood of writes. If we flood write to the Disk, the Disk will explode quickly, also the value of the same key often changes or new keys flood in. However, the LSM tree needs to maintain sorted data, which will result in having to search many layers to get the correct value for multiple queries. Therefore, LSM has introduced compaction, which can continuously __integrate__ the written result into sstable, and maintain the result as much as possible to update and sorted state. This is called "__Compaction__"



### Compactions illustrated

How to do compaction has actually become the core problem in the LSM tree architecture, and it is also the core balance of efficiency and cost. In order to facilitate understanding, we can assume the two extremes of compaction,  [This article](https://stratos.seas.harvard.edu/files/stratos/files/dostoevskykv.pdf) has a good explanation (by the way, I strongly recommend you to read this Lab article, very good)

![compare_0](https://github.com/zhangruiskyline/system/blob/main/images/compare_0.png)

#### First approach(no compactions)


The first extreme way (the upper left log scheme) is that no compaction is done at all. All data is written to the Log and then poured into the SStable. This approach, of course, has the lowest update cost and no compaction at all, just write directly to Disk. But once you want to query or update a K/V, it will cost a lot. There is no sort and no old values ​​are removed. You can only check linearly and slowly, so the efficiency is very low, and if there is no compaction, the data will be written along with it. Linear growth, faster Disk will explode.

#### Second approach(Always sort)

The second extreme way(sorted array scheme at the bottom right) is that every time a new record is written, compaction will be done immediately, and the entire data will always be updated to the latest value and sorted state, so that the query efficiency is the highest, directly according to the index, but every time The entire data need to be adjusted for every write, and sort and update are too expensive. when Write KPS becomes high, the machine cannot support such high load

To make an example, the two options are like the two extremes of cleaning up the house: the first is that you are super lazy and don't clean up at all. Things are left at random. When you throw sth new you don't care anything, but you can only find them slowly in a mess. The second is the cleanliness habit, keeping tidy up all the time, the house is completely uniform and organized, but there is no time to do other things.

Therefore, the actual DB of LSM Tree is a compromise between the two schemes. Like most of us, we may  clean up "from time to time".  LSM also needs to do compaction "from time to time". So what kind of "from time to time" is appropriate? Two basic compaction strategies will be introduced below: __level vs size/tier__


## Leveling Compaction


Before we dive into leveling compaction, we need to understand one importation term in LSM tree. __Run__. I think the best explaination comes from wiki:

> Each run contains data sorted by the index key. A run can be represented on disk as a single file, or alternatively as a collection of files with non-overlapping key ranges.
Key points: What conditions need to be met for run? The first sorted, the second non-overlapping key ranges
	
In Summray, there should be two most important features in run

1. Sorted

2. non-overlapping key ranges

This [RocksDB introduction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction) describes the practice of Leveling compaction in great detail. As shown in the figure below, a multi-level SStable is maintained in Disk, and each layer maintains a single Run. 

![leveling1](https://github.com/zhangruiskyline/system/blob/main/images/compaction_1.png)

![leveling2](https://github.com/zhangruiskyline/system/blob/main/images/compaction_2.png)

The level is a sorted run because keys in each SST file are sorted (See Block-based Table Format as an example). To identify a position for a key, we first binary search the start/end key of all files to identify which file possibly contains the key, and then binary search inside the file to locate the exact position. In all, it is a full binary search across all the keys in the level.

Compaction triggers when number of L0 files reaches level0_file_num_compaction_trigger, files of L0 will be merged into L1. Normally we have to pick up all the L0 files because they usually are overlapping:

![leveling_l0](https://github.com/zhangruiskyline/system/blob/main/images/level_1.png)

After the compaction, it may push the size of L1 to exceed its target:


![leveling_l1](https://github.com/zhangruiskyline/system/blob/main/images/level_2.png)

And so on so forth to more levels.

To summarize briefly, the goal of Level compaction is to maintain one data sorted run at each level, so each level can be compacted with the next level, and it is likely to be compacted by the previous level. The advantage of this is that the compaction between levels can be done by multithreading (except memory to level0), which improves efficiency. The default used by RocksDB is Leveling compaction. Below is a illustration how it is done in multi threading


![leveling_multithreading](https://github.com/zhangruiskyline/system/blob/main/images/level_3.png)


## Size/Tier Compaction


In addition to leveling compaction, another common compaction is Size or Tier compaction. How is it done? There is a very classic description in [this article](https://www.scylladb.com/2018/01/17/compaction-series-space-amplification/) as shown below


![size_1](https://github.com/zhangruiskyline/system/blob/main/images/size_1.png)

As usual, memtables are periodically flushed to new sstables. These are pretty small, and soon their number grows. As soon as we have enough (by default, 4) of these small sstables, we compact them into one medium sstable. When we have collected enough medium tables, we compact them into one large table. And so on, with compacted sstables growing increasingly large.

We have seen that this compaction method can guarantee that each sstable is sorted, but cannot guarantee that there is only one Run for each layer? why? let's revisit how to define a __run__: a collection of files with non-overlapping key ranges. The right is this none-overlap key range. This level compaction can be guaranteed. Size or Tier compaction can’t (in fact, to be more precise, Tier compaction, the run can be guaranteed on the largest layer). Cassandra defaults to Tier's compaction

## Leveling vs Tier Compaction

we can go back to the [This article](https://stratos.seas.harvard.edu/files/stratos/files/dostoevskykv.pdf). And check the below example 

![compare](https://github.com/zhangruiskyline/system/blob/main/images/compare.png)

* In size/Tier compaction, each sstable, that is, the array in each box is sorted. This is no problem, but in level 2, [1,3,4,7], [2,5,6,8] Can not form a Run, because the key range is overlap. So there is no guarantee that there is only one Run for each level. For example, if I search for 5, then 5 meets the conditions in the two ranges<2,8>,<1,7>, and it is impossible to use Binary search to determine which sstable it is in.

* Leveling compaction, however, can guarantee __Run__.  Note that when level 1 merges to level 2, the data of level 1 and the data of level 2 will be compacted again, forming the only "Run", which can be found faster when searching. Of course the price paid is the need for more frequent compaction.


I hope everyone has an understanding of leveling and Tier compaction until now, and you can look at the coordinates in the first picture again. The default RocksDB  Leveling compaction and   default Cassandra Tier compaction are located in different space in curve, and you can have a better understanding why.


### Amplification 

Once we have this illustrated idea, we can check some more formal evulation way of this problem. you can refer [this article](http://smalldatum.blogspot.com/2015/11/read-write-space-amplification-b-tree.html) and [this article](http://smalldatum.blogspot.com/2015/11/read-write-space-amplification-pick-2_23.html) also

#### Write amplification

Write amplification means that the same record needs to be written to Disk multiple times. The larger the number, the more Disk writes. (we use Clean up the room example mentioned before, it means takes more times to clean up the room). Obviously leveling requires more frequent non-stop compaction to ensure that each level has only one Run, so its Write Amplifier is larger

#### Read amplification

Read amplification means how many disk reads need to be read for a record. The larger the number, the more Disk Reads (or use the clean room to understand, which means how many times you need to dig through the cabinets to find what you need in the room). Leveling compaction is more frequent than updating compaction each time, ensuring that each level has a unique Run, so Read Amplifier has advantages over Tier compaction. You can refer to the difference between the query "5" in our picture above

#### Size amplification

Space Amplifier means that in order to reach the final ideal state, how much disk space is needed to put the intermediate result of temporary compaction. The larger the number, the more Disk overhead (we use same clean room to understand, which means that more utility rooms are needed to temporarily stack things). 


shows the difference between the two very well. Diligent leveling compaction is relatively smaller in space Amplifier


## Last but not least

I hope you enjoy this article so far. Last but not least, hope it is the journey for all of us to explore more.  Exactly as my favorite quote:

* Now this is not the end. It is not even the beginning of the end. But it is, perhaps, the end of the beginning. --Winston Churchill







