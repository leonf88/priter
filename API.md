Besides the job configurations specified like Hadoop job. PrIter asks users to specify Activator and Updater instead of Mapper and Reducer. Users should specify the priority class in addition to key class and value class.

# Configure Job #

  * `void JobConf.setActivatorClass(Class<? extends Activator>);`

> set the activator class


  * `void JobConf.setUpdaterClass(Class<? extends Updater>);`

> set the updater class


  * `void JobConf.setPriorityClass(Class<? extends Writable>);`

> set the priority class

# User-Defined Functions #

## Data Partition ##

  * `Distributor.partition(String input_path, String output_path, int num_partitions, Class<? extends Writable> keyClass, Class<? extends Partitioner> partititonerClass)`

> before executing the main iterative computation, users should first distribute the data to workers. `Distributor` is provided in PrIter framework, which partitions the input data to a number of partitions distributed to workers. These data partitions are processed by these workers distributively. The keyClass is the data item index, which could be IntWritable or Text, or etc. The partitionerClass tells system how to partition the data, which will also be used for finding the destination workers during shuffling.

## Along with Activator Interface ##

Activator interface is required to be implemented. The Activator activates operations on the nodes in the Priority Queue.

`public interface Activator<K, P extends WritableComparable, V> extends JobConfigurable {`

  * `void initStarter(InputPKVBuffer<V> starter) throws IOException;`

> to be invoked before iteration starts, init the starter for iterative compuattion. In other words, initialize the input Priority Queue, the nodes in which are activated to start the iteration.

  * `void activate(IntWritable nodeid, V value, OutputCollector<IntWritable, V> output, Reporter reporter);`

> to be invoked when activating node with value, the neighbors of node will be affected, users specify the values sent to neighbors to realize their algorithm logic

  * `void iterate();`

> to be invoked after emitting all the results to neighbors after each iteation, implement some tracke codes

`}`
## Along with Updater Interface ##

Updater interface is required to be implemented. The Updater updates the StateTable entries according to the algorithm logic.

`public interface Updater<K, P extends Valueable, V extends Valueable> extends JobConfigurable {`

  * `void initStateTable(OutputPKVBuffer<P, V> stateTable);`

> to be invoked before iteration starts, init StateTable

  * `V resetiState();`

> to be invoked when node record reset, reset the node's iState

  * `P decidePriority(IntWritable nodeid, V iState);`

> to be invoked when updating statetable, decide node's execution priority (based on iState).

  * `P decideTopK(IntWritable nodeid, V cState);`

> to be invoked when extracting top k result, decide node's priority for online top k query (based on cState)

  * `void updateState(K nodeid, V iState, OutputPKVBuffer<K, P, V> stateTable, Reporter reporter) throws IOException;`

> to be invoked when updating StateTable, users specify the update rule to realize their algorithm logic

  * `void iterate();`

> to be invoked after emitting all the results to neighbors after each iteation, implement some tracke codes

`}`