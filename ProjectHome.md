## What is PrIter? ##
The PrIter project started at University of Massachusetts Amherst from 2011. PrIter is a modified version of Hadoop MapReduce framework that supports prioritized iterative computation, which support a large collection of iterative algorithms, including pagerank and shortest path. PrIter runs on a cluster of commodity PCs or in Amazon EC2 cloud. It
ensures faster convergence of iterative process by reorganizing the update order of data items. Priter also supports online queries and generates top-k result snapshot every period of time. For details, please read our [paper](http://rio.ecs.umass.edu/~yzhang/papers/socc11-final71.pdf) accepted in [SOCC 2011](http://socc2011.gsd.inesc-id.pt/).


Currently, PrIter is just a prototype. It is better for understanding the priority idea than for production usage. Any feedback is appreciated and we welcome your involvement in this project. If you have any question, please contact me (yanfengzhang@ecs.umass.edu).

Recently, the file-based PrIter is implemented, which maintains data in files instead of in memory. This ensures PrIter to scale much larger data sets. Memory-based PrIter is for better performance, while file-based PrIter is for better scalability. But notice that, we have tested that the file-based PrIter is only 2 times slower than the memory-based PrIter for Pagerank computation.

## Ongoing Works ##
  * Efficient fault tolerance and load balancing.
  * C++ version, which is better for memory-intensive computation than Java. Please see [Maiter](http://code.google.com/p/maiter/).

## News ##
  * PrIter-0.2 is released, it supports storing data in files instead of in memory, and can provide very good performance (only 2 times slower than memory-based priter), please refer to the pagerank example code for file-based priter application's implementation. I will update the introduction to the new API soon. (27/Feb/2012).
  * File-based PrIter is implemented (22/Feb/2012).
  * The first version of PrIter is released (25/Oct/2011).
  * Sample application scripts are added (24/Oct/2011).
  * Clustering algorithms (pairwise cluster and K-means cluster) are implemented in PrIter (13/Aug/2011).
  * PrIter paper is accepted by SOCC 2011 (11/Jul/2011).
  * Katz metric algorithm is implemented in PrIter (4/May/2011).
  * Asynchronous activation for frameworks (not algorithm asynchronism) implemented by self trigger (3/Apr/2011).

## Getting Started ##
PrIter is implemented based on [Hadoop 0.19.2](http://hadoop.apache.org/) and [HOP](http://code.google.com/p/hop). The Hadoop jobs can also run in PrIter with the same code.

  1. Download [hadoop-priter-0.1.tar.gz](http://priter.googlecode.com/files/hadoop-priter-0.1.tar.gz).
  1. Unpack this tar and deploy PrIter cluster using one or more commodity PCs. The deployment of PrIter in a distributed environment is the same as Hadoop's deployment. You can refer to [Hadoop Quick Start instructions](http://hadoop.apache.org/common/docs/current/), if you've never used Hadoop. There are a few notes:
    * Modify {priter\_path}/conf/masters to set master and {priter\_path}/conf/slaves to set slave workers
    * There are a series of parameters in {priter\_path}/conf/hadoop-site.xml, where you should set the jobtracker server and namenode server. And you can tune the memory related parameters according to your hardware configuration to maximize PrIter performance.
    * Set a swap file or swap partition since PrIter may report memory related errors if your PCs are memory limited.
    * The maximum number of map/reduce tasks in each worker should be set big enough to allow task migration.
  1. Go to {priter\_path}/apps directory, the shell scripts of four sample applications are provided. Here we take the pagerank script as an example:
    * Download a real graph from [http://rio.ecs.umass.edu/~yzhang/data/](http://rio.ecs.umass.edu/~yzhang/data/) or generate a synthetic unweighted graph by running gendata.sh.
    * If you download a real graph, upload the local data to HDFS:
> > > `{priter_path}/bin/hadoop dfs -put {local_data_path} {HDFS_path}`
    * If you want to generate synthetic data, the gendata.sh script will help you generate synthetic lognormal graphs distributively according to some lognormal parameters on node degree and link weight:
> > > `{priter_path}/bin/hadoop jar {priter_path}/hadoop-priter-0.1-examples.jar gengraph -p <num_paritions> -n <num_nodes> -t <graph_type> -degree_mu 0.5 -degree_sigma 2 -weight_mu 0.5 -weight_sigma 1 {HDFS_output_path}`
    * Run pagerank:
> > > `{priter_path}/bin/hadoop jar {priter_path}/hadoop-priter-0.1-examples.jar pagerank -p <num_partitions> -k <topk> -q <queue_size> -i <snapshot_interval> -t <termination_difference> {HDFS_input_path} {HDFS_output_path}`

> > The pagerank code can be found in {priter\_path}/src/examples. It first distributes the graph data to workers by an MapReduce job and then performs pagerank by prioritized iteration. Every period of time _snapshot\_interval_, PrIter outputs a top-k snapshot on HDFS (_HDFS\_output\_path_). The computation terminates when the difference between two iteration progresses measured by two consecutive snapshot time points are smaller than _termination\_difference_. Users can set _queue\_size_ to control the prioritization degree, which is a balance between priority benefit and priority overhead.
    * Run other applications:
> > More applications scripts can be found in {priter\_path}/apps, including shortest path, adsorption, katz metric. Their source codes are included in the example source dir.

Something that should be noticed when running PrIter.
  * It is normal that the job progress is stuck at 0%. Because the iterations are within a single job, we perform the termination check by measuring the distance between two snapshot results (terminate when the distance is smaller than a threshold, the number of iterations is unknown), it is difficult to estimate the iteration progress. So we skip the progress monitor implementation. You can see the progress by checking the generated snapshots on HDFS or tracking the task logs (or do something in the `iterate()` interface).
  * The setting of the snapshot interval parameter (set by -i) is important. PrIter performs termination check along with generating the snapshot, so the snapshot interval is the same as the termination check interval. Suppose we measure the distance between two consecutive snapshots, snapshot1 and snapshot2. If |snapshot1-snapshot2|<terminate threshold (set by -t), the system terminate the iteration. Therefore, the users should set snapshot interval (-i) considering the setting of terminate threshold (-t). If -i is high, -t should be high. if -i is very small, the iteration could be stopped very quickly, because the change between two snapshots is small (smaller than terminate threshold).


The following pages provide further implementation details that help you know how to program in PrIter:
### [An brief introduction of PrIter](PrIterIntroduction.md) ###
### [PrIter API](API.md) ###
### [PrIter system parameters](SystemParameters.md) ###
### [PageRank code](Impl_Ex2_PageRank.md) ###

More can be found in [Wikipage](http://code.google.com/p/priter/w/list).