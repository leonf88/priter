# General Framework Parameters #

| **property** | **type** | **default** | **description** |
|:-------------|:---------|:------------|:----------------|
| "priter.job" | boolean  | true        | turn on priter framework, otherwise it does normal MapReduce job |
| "mapred.jobtracker.taskScheduler" | string   | org.apache.hadoop.mapred.PrIterTaskScheduler | use PrIter's task scheduler for assign MRPairs|

## Fault Tolerance & Load Balancing ##
| "priter.checkpoint" | boolean | true | turn on checkpoint for the support of fault tolerance and load balancing|
|:--------------------|:--------|:-----|:------------------------------------------------------------------------|
| "priter.checkpoint.frequency" | int     | 10   | the StateTable checkpoint dump frequency for fault tolerance and load balancing |
| "priter.task.migration.threshold" | long    | 20000 | the time difference (ms) threshold to the average to perform task migration |
| "priter.task.restart.threshold" | long    | 40000 | how long (ms) is it should we wait to ensure some task is failed after receiving last task's iteration completion report|
| "priter.task.checkrollback.frequency" | long    | 2000 | the frequency for a task to check if should do roll back                |


---


---

# Application Specific Parameters #

## PrIter Job ##
| **property** | **type** | **default** | **description** |
|:-------------|:---------|:------------|:----------------|
| "priter.job.priority" | boolean  | true        | turn on prioritized iteration, otherwise it performs normal iteration on top of PrIter |
| "priter.job.priorityclass" | class    | ---         | the priority data type |

## Input Graph Information ##
| "priter.graph.partitions" | int | 2 x (# of tasktrackers) | the number of graph partitions |
|:--------------------------|:----|:------------------------|:-------------------------------|

## Onlie Query Support ##
| "priter.snapshot.interval" | long | 10000 | the time interval (ms) of snapshot generation for online query |
|:---------------------------|:-----|:------|:---------------------------------------------------------------|
| "priter.snapshot.topk"     | integer | 1000  | specify the number of interested top records                   |
| "priter.snapshot.topk.scale" | integer | 4     | (priter.snapshot.topk.scale)x(priter.snapshot.topk)/(priter.graph.partitions) number of top records on each MRPair will be sent to merge worker to merge top k result online. |

## Activation Queue Setting ##
| "priter.queue.portion" | float | --- | the activation queue length (portion of total nodes) |
|:-----------------------|:------|:----|:-----------------------------------------------------|
| "priter.queue.length"  | integer | --- | the activation queue length                          |
| "priter.queue.uniqlength" | integer | --- | the activation queue length (unique queue, with elements are unique) |

## Iteration Termination ##
| "priter.stop.difference" | float | --- | stop iteration by check difference between two continuous iteration progresses |
|:-------------------------|:------|:----|:-------------------------------------------------------------------------------|
| "priter.stop.maxiteration" | integer | --- | stop iteration when it iterates max rounds                                     |
| "priter.stop.maxtime"    | long  | --- | stop iteration when its iterates long enough                                   |