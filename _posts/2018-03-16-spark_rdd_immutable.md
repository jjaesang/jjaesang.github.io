---
layout: post
title:  "RDD가 Immutable한 이유 "
date: 2019-03-16 18:44:02
categories: Spark
author : Jaesang Lim
tag: Spark
cover: "/assets/spark.png"
---

# I don't understand the reason behind Spark RDD being immutable.

1. Answer 1)
- Immutable data is always safe to share across multiple processes as well as multiple threads.
- Since RDD is immutable we can recreate the RDD any time. (From lineage graph).
- If the computation is time-consuming, in that we can cache the RDD which result in performance improvement.

2. Answer 2)
- Apache Spark on HDFS, MESOS or Local mode distributes and store transformation data in the form of RDD (Resilient Distributed DataSets).
- RDDs are not just immutable but a deterministic function of their input. That means RDD can be recreated at any time.This helps in taking advantage of caching, sharing and replication. RDD isn't really a collection of data, but just a recipe for making data from other data.
- Immutability rules out a big set of potential problems due to updates from multiple threads at once. Immutable data is definitely safe to share across processes.
- Immutable data can as easily live in memory as on disk. This makes it reasonable to easily move operations that hit disk to instead use data in memory, and again, adding memory is much easier than adding I/O bandwidth.
- RDD significant design wins, at cost of having to copy data rather than mutate it in place. Generally, that's a decent tradeoff to make: gaining the fault tolerance and correctness with no developer effort worth spending disk memory and CPU on.

3. Answer 3 with quora
- It is easy to share the Immutable data safely among several process. Basically, due to updates from multiple threads at once, Immutability rules out a big set of potential problems.
- Immutable data can as easily live on memory as on disk. This makes it easy move operations from the that hit disk to instead use data in memory. adding memory is much easier then adding i/o bandwidth.
- Basically, RDDs are not just immutable but also deterministic function of their input. That means RDD can be recreated at any time. It helps in leverages the advantage of caching, sharing and replication. It isn’t really a collection of data but also a way of making data from other data.
- If the computation is time-consuming, in that we can cache the RDD which result in performance improvement.

4. 그냥 주구절절 답
- An RDD is an abstraction of a collection of data distributed on various node. An RDD is characterized by his lineage, the different operations needed to obtain this RDD from others. Also Spark is based on Scala data structures, and functional transformations of data. For these reasons spark uses immutable data structure.
- RDDs composed of collection of records which are partitioned. Partition is basic unit of parallelism in a RDD, and each partition is one logical division of data which is immutable and created through some transformations on existing partitions.Immutability helps to achieve consistency in computations.
- Users can define their own criteria for partitioning based on keys on which they want to join multiple datasets if needed.
- When it comes to iterative distributed computing, i.e. processing data over multiple jobs in computations such as  Logistic Regression, K-means clustering, Page rank algorithms, it is fairly common to reuse or share the data among multiple jobs or it may involve multiple ad-hoc queries over a shared data set.This makes it very important to have a very good data sharing architecture so that we can perform fast computations.

참고 ) [스파크 관련 Data-flair 블로그](https://data-flair.training/blogs/fault-tolerance-in-apache-spark/)
참고 ) [외국인 아저씨 LinkedIn 블로그](https://www.linkedin.com/pulse/my-note-spark-rdd-manindar-g/)

- 위 내용 기반으로 내 스스로의 답을 만들자 ! 이번주에 꼭
