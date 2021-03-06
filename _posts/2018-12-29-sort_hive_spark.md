---
title: "基于hive/spark的大规模数据全局排序的几个想法"
categories:
- 大数据
- 随笔
tags:
- 算法
toc: true
---

{% include lib/mathjax.html %}

 ## 1. 问题

最近遇到一个关于大数据量全排的问题：约16亿条数据，数据倾斜较为严重（某些值出现的频率为千万级别）需要确切的知道每条数据的排序（相同值的排名相同，并且排名是跳跃的，也就是hive中rank函数的功能），直接全排显然行不通，mr的瓶颈在于数据全部进一个reduce，效率极低，spark则内存不足，导致任务崩溃，如何效率更高的达到目标？

## 2.几个思路

### 2.1 基于MR TotalOrderPartitioner

**想法**：通过随机抽样得到分段的间隔点，基于间隔点数据以及 TotalOrderPartitioner 将数据进行分区，在每个分区内部进行全排，再将区间拼接起来得到全局排序的数据。

**解决步骤**：

1. 使用 TABLESAMPLE(BUCKET m OUT OF n ON RAND() )对数据进行随机抽样(通过计算总条数确定n来确保抽样出来的结果条数)，假设随机抽取出来的数据条数为50万，进行全排后，等间隔抽取100条并做去重，得到间隔点的数据文件 _partition.lst ；

2. 设置 hive 做 mr 的分区方式，并添加数据文件至分布式存储，mr会自动寻找文件并根据文件划分分区，再对tbl1 的 sk 字段直接进行排序。（cluster by = distribute by + sort by）；

   ```
   set mapred.reduce.tasks=n;
   set hive.mapred.partitioner=org.apache.hadoop.mapred.lib.TotalOrderPartitioner;
   add file hdfs:// ../_partition.lst;
   
   select * from tbl1 cluster by sk;
   ```

3. 对排序好的数据文件，编写 java 方法遍历一遍文件，依照逻辑在每条数据后边加上字段 ‘排名’（该步骤的性能极快），达到目标。

**瓶颈**：虽然通过抽样的方式，得到大概合理的分段点，将一个大的reduce任务，划分为多个小的reduce任务，但是某些单个值的数据倾斜问题依旧无法解决，单个值的数据依旧会进入同一个reduce，导致某些reduce任务依旧很大。spark 中有类似思路的方法，RangePartitioner。



### 2.2 基于对 sk 做去重并 count，再计算得到

**想法**：对 sk 进行去重并计算每个 sk 出现的次数 count，再对去重后的 sk 进行全排，根据每个sk出现的位置以及次数，可以累加计算得到每个 sk 的排名。

**解决步骤**：

1. 对 sk 进行去重并计算每个 sk 出现的次数 count；

   ```
   insert overwrite table tbl2
   select sk,count(*) from tbl1 group by sk;
   ```

2. 对去重后的 sk 进行全排，可以得到类似下表的数据;

   | sk   | count |
   | ---- | ----- |
   | sk_1 | n_1   |
   | sk_2 | n_2   |
   | sk_3 | n_3   |
   | sk_4 | n_4   |
   | sk_5 | n_5   |

   ```
   select * from tbl2 order by sk;
   ```

3. 根据每个sk出现的位置以及次数，累加计算得到每个 sk 的排名：

   这一步有几种实现方法

   - 使用 hive 的窗口函数直接得到排名：

     ```
     select *,sum(count) over(order by sk) from tbl2;  ##需要分区内累加时可加上partition by
     ```

   - 编写udf，udf的实现逻辑为：对 tbl2 的文件遍历计算得到每个 sk 准确的排名,存为map，输入sk便可输出对应的排名（可直接对tbl1或者tbl2使用）；

   由此可以得到：

   | sk   | count | rank              |
   | ---- | ----- | ----------------- |
   | sk_1 | n_1   | 1                 |
   | sk_2 | n_2   | 1+n_1             |
   | sk_3 | n_3   | 1+n_1+n_2         |
   | sk_4 | n_4   | 1+n_1+n_2+n_3     |
   | sk_5 | n_5   | 1+n_1+n_2+n_3+n_4 |

4. 如果上一步使用窗口函数实现，则需将 tbl1 与 tbl2 关联；

   如果上一步使用udf实现，则既可以直接对 tbl2 作用，再与 tbl1 关联;也可以直接对 tbl1 作用，得到全部的排名。

**瓶颈**：1、对 sk 去重过后的数量级不稳定，如果去重过后 sk 在千万级，全局排序尚可，若仍为亿级，则去重来降低排序难度以及解决数据倾斜情况的目的则没有达到；

2、编写 udf 的可拓展性不高，但是直接使用hive自带的窗口函数时，发现在 sum() over(order by) 时的时间复杂度应该是 \\(n^2\\)，在测试环境下，当数量级达到10万时，便会耗费大量时间，若为千万级别，应该会跑不出来；



## 3. 基于以上两种思路的结合

**想法1**：对 sk 进行按顺序进行分段，为防止某个值出现的频率极大，可使用思路2对 sk 进行去重并count，使用topN 对前N个单个值进行处理，这些单个值每个单独作为一组（段），且内部不需要排序，对于剩下的段，对全表给每一段打上标签 sksk（也就是使用 case when 新增一列），然后 实现 rank() over(partition by sksk order by sk) 的逻辑并计算每一个 sksk 的count，从而得到了每个大段的排名，以及大段内部每个 sk 的排名，从而可以计算得到每个 sk 的排名。

**解决步骤**：待商榷。

**瓶颈**：面临如何选取合理的分段点，使得每一段内数据相对均匀。虽然去除了最大的几个单个值，但是剩余的数据分布是否稳定，选取固定的分段点必定不合理。根据抽样选取分段点，相对合理。是否有更好的方法？并且实现起来相对较为复杂。

**想法2**：对思路2中去过重之后的 sk进行全排时，使用思路1中的方法，进行抽样，分区，TotalOrderPartitioner，从而突破思路2中去重全排的瓶颈，且这一步不会有数据倾斜。而在spark中，由于去重过后数据完全均匀，所以采用直接全排效率极高（默认使用RangePartitioner），测试环境下亿级别的数据量也在一分钟之内完成。

**解决步骤**：与2类似，将1中的方法（MR需使用TotalOrderPartitioner 或 spark默认RangePartitioner）替换掉第2步的全排。

**瓶颈**：目前思考来看，想法2的思路较为稳定，但是实验步骤约等于思路1加思路2。目前验证的情况来说，性能最好最稳定。