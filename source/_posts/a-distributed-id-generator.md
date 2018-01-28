---
title: 一个分布式全局唯一id产生器(python)
date: 2017-01-15 00:33:36
tags:
---
最近使用graphx的时候需要对图中的每个顶点进行编号，这个在非分布式的程序中也许不是一个问题，采用一个单独的变量进行累加即可。可是在分布式系统中，数据存储在hdfs上，如果使用传统的方法，需要将所有数据加载到一台机器上，当数据量大的时候，网络和存储的开销是很大的，比如每小时需要处理10G的数据，对每一行进行全局唯一的编号。

Graphx的顶点编号只支持64位，不像giraph支持字符串类型的编号（可用md5编号），也不像neo4j支持namespace下的编号。不过现成的方案也是有的，我下面就搬出来解释一下。

采用时间戳，工作节点序号，分布式累加器的组合，可以完成这个巨大的任务。时间戳保证了这些编号在时间上是有序的，先出现的顶点编号小于后出现的顶点编号（比如一个订单的id），工作节点序号保证了分布式环境下的编号唯一性，累加器保证了同一个工作节点在同一个毫秒之内，对顶点的编号也是唯一的。64位的编码，为了不让编号出现负数的情况，最高位要为0，所以实际可用的是63位（应对我目前的需求是绰绰有余了）

首先我们有多少个处理节点可以同时工作，spark中可以指定分区数量进行数据处理，闭着眼随便喊个数字，256个应该够了。那么对工作节点的位数可以大致定位8位（总共能使用256个worker id),然后就是确定单个工作节点每毫秒理论上最多处理多少个编号，先盲目的定一个12位吧，这样我们每毫秒每个工作节点可以处理4096条数据（乘上工作节点的数量，每毫秒我们的系统可以处理100多万条数据，看起来不赖吧）。最后我们剩下了43位的时间戳，43位的时间戳可以表示270多年的时间。

上面提到的三个字段，都可以根据实际情况适当调整，如果对时间比较敏感，可以缩短累加器的长度；如果机器资源有限，也可以减少分布式工作节点的长度。目前我用10台机器的集群，总共40多G内存，对2000多万的数据进行编号，大概总共需要不到10分钟的时间（spark的运行加载，数据的io比较费时）。整体来说还是可以接受的。

Spark中指定分区数量，并且获取每个分区的id,可以使用repartition和mapPartitionsWithIndex这两个方法。

```
# -*- coding: utf-8 -*-

import time

epoch = 1479916800000L

worker_id_bits = 8L

max_worker_id = -1L ^ -1L << worker_id_bits

sequence_bits = 12L

worker_id_shift = sequence_bits

timestamp_left_shift = sequence_bits + worker_id_bits

sequence_mask = -1L ^ -1L << sequence_bits 
class IdWorker: 
    def __init__(self, worker_id): 
        self.sequence = 0L 
        self.last_timestamp = -1L 
        if worker_id > max_worker_id or worker_id < 0:
            raise Exception("worker Id can't be greater than %d or less than 0" % max_worker_id)

        self.worker_id = worker_id

    def time_gen(self):

        ts = long(time.time()*1000L)

        return ts

    def til_next_millis(self, last_timestamp):

        ts = self.time_gen()

        while ts <= last_timestamp:

            ts = self.time_gen()

        return ts

    def get_next_id(self):

        ts = self.time_gen()

        if self.last_timestamp == ts:

            self.sequence = (self.sequence + 1) & sequence_mask

            if self.sequence == 0:

                ts = self.til_next_millis(self.last_timestamp)

        else:

            self.sequence = 0

        if ts < self.last_timestamp:

            raise Exception("Clock moved backwards. Refusing to generate id for %d milliseconds"

                            % self.last_timestamp - ts)

        self.last_timestamp = ts

        next_id = (ts - epoch << timestamp_left_shift) | (self.worker_id << worker_id_shift) | self.sequence

        return next_id

if __name__ == "__main__":

    worker2 = IdWorker(2)

    worker3 = IdWorker(3)

    print max_worker_id

    for i in range(10):

        print worker2.get_next_id()

        print worker3.get_next_id()
```