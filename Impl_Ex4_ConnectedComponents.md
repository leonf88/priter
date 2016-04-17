# Introduction #
Connected Components is an algorithm for finding the connected components in large graphs. The main idea is as follows. For every node  in the graph, it is associated with a component id, which is initially set to be its own node id. In each iteration, each node propagates its current component id to its neighbors. Then the component id of each node, is set to be the maximum value among its current component id and the received component ids. Finally, no node in the graph updates its component id where the algorithm converges, and the connected nodes have the same component id.

In the prioritized Connected Components algorithm, we let the nodes with larger component ids propagate their component ids rather than letting all the nodes do the propagation together. In this way, the unnecessary propagation of the small component ids are avoided since those small component ids will probably be updated with larger ones in the future. The prioritized Connected Components algorithm can be described using MapReduce model as follows.

# Details #

**_Activator_**
```
public class ConnectComponentActivator extends MapReduceBase implements
		Activator<IntWritable, IntWritable, IntWritable> {

	private String subGraphsDir;
	private int kvs = 0;
	private int iter = 0;
	
	//graph in local memory
	private HashMap<Integer, ArrayList<Integer>> linkList = new HashMap<Integer, ArrayList<Integer>>();

	
	@Override
	public void configure(JobConf job) {
		int taskid = Util.getTaskId(job);
		loadGraphToMem(job, taskid);
	}
	
	private synchronized void loadGraphToMem(JobConf conf, int n){
		subGraphsDir = conf.get(MainDriver.SUBGRAPH_DIR);
		Path remote_link = new Path(subGraphsDir + "/part" + n);
		
		FileSystem hdfs = null;
	    try {
			hdfs = FileSystem.get(conf);
			FSDataInputStream in = hdfs.open(remote_link);
			BufferedReader reader = new BufferedReader(new InputStreamReader(in));
			
			String line;
			while((line = reader.readLine()) != null){
				int index = line.indexOf("\t");
				if(index != -1){
					String node = line.substring(0, index);
					
					String linkstring = line.substring(index+1);
					ArrayList<Integer> links = new ArrayList<Integer>();
					StringTokenizer st = new StringTokenizer(linkstring);
					while(st.hasMoreTokens()){
						links.add(Integer.parseInt(st.nextToken()));
					}
					
					this.linkList.put(Integer.parseInt(node), links);
				}
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	
	@Override
	public void initStarter(InputPKVBuffer<IntWritable, IntWritable> starter)
			throws IOException {
		for(int k : linkList.keySet()){
			starter.init(new IntWritable(k), new IntWritable(k));
		}
	}
	
	@Override
	public void activate(IntWritable key, IntWritable value,
			OutputCollector<IntWritable, IntWritable> output, Reporter report)
			throws IOException {
		kvs++;
		report.setStatus(String.valueOf(kvs));
		
		int node = key.get();
		if(linkList.get(node) == null) return;
		for(int linkend : linkList.get(node)){
			output.collect(new IntWritable(linkend), value);
		}
	}

	@Override
	public void iterate() {
		System.out.println((iter++) + " passes " + kvs + " activations");
	}
}

```

**_Updator_**
```
public class ConnectComponentUpdater extends MapReduceBase implements
		Updater<IntWritable, IntWritable, IntWritable> {

	private int workload = 0;
	private int iterate = 0;
	
	@Override
	public void initStateTable(OutputPKVBuffer<IntWritable, IntWritable, IntWritable> arg0) {

	}

	@Override
	public void iterate() {
		iterate++;
		System.out.println("iteration " + iterate + " total parsed " + workload);
	}
	
	@Override
	public IntWritable resetiState() {
		return new IntWritable(Integer.MIN_VALUE);
	}

	@Override
	public IntWritable decidePriority(IntWritable key, IntWritable iState) {
		return new IntWritable(iState.get());
	}

	@Override
	public IntWritable decideTopK(IntWritable key, IntWritable cState) {
		return new IntWritable(cState.get());
	}
	
	@Override
	public void updateState(IntWritable key, Iterator<IntWritable> values,
			OutputPKVBuffer<IntWritable, IntWritable, IntWritable> buffer, Reporter report)
			throws IOException {
		workload++;		
		report.setStatus(String.valueOf(workload));
		
		int max_id = values.next().get();

		synchronized(buffer.stateTable){
			PriorityRecord<IntWritable, IntWritable> pkvRecord;	
			if(buffer.stateTable.containsKey(key)){
				pkvRecord = buffer.stateTable.get(key);
	
				int cState = pkvRecord.getcState().get();
				if(max_id > cState){
					buffer.stateTable.get(key).getiState().set(max_id);
					buffer.stateTable.get(key).getcState().set(max_id);
					buffer.stateTable.get(key).getPriority().set(max_id);
				}
			}else{
				pkvRecord = new PriorityRecord<IntWritable, IntWritable>(
						new IntWritable(max_id), new IntWritable(max_id), new IntWritable(max_id));
				buffer.stateTable.put(new IntWritable(key.get()), pkvRecord);
			}
		}
	}
}
```

**_Main_**
```
public class ConnectComponent extends Configured implements Tool {
	private String input;
	private String output;
	private String subGraphDir;
	private int partitions;
	private int topk;
	private int exetop;
	private long snapinterval = 5000;
	private long stoptime;
	
	
	private int conncomp() throws IOException{
	    JobConf job = new JobConf(getConf());
	    String jobname = "connect component";
	    job.setJobName(jobname);
       
	    job.set(MainDriver.SUBGRAPH_DIR, subGraphDir);
	    if(partitions == 0) partitions = Util.getTTNum(job);			//set default partitions = num of task trackers
	    
	    FileInputFormat.addInputPath(job, new Path(input));
	    FileOutputFormat.setOutputPath(job, new Path(output));
	    job.setOutputFormat(TextOutputFormat.class);

	    //set for iterative process   
	    job.setBoolean("priter.job", true);
	    job.setInt("priter.graph.partitions", partitions);			//graph partitions
	    job.setLong("priter.snapshot.interval", snapinterval);		//snapshot interval	
	    job.setInt("priter.snapshot.topk", topk);					//topk
	    job.setInt("priter.queue.uniqlength", exetop);				//execution queue
	    job.setLong("priter.stop.maxtime", stoptime);				//termination check

	    
	    job.setJarByClass(ConnectComponent.class);
	    job.setActivatorClass(ConnectComponentActivator.class);	
	    job.setUpdaterClass(ConnectComponentUpdater.class);
	    job.setMapOutputKeyClass(IntWritable.class);
	    job.setMapOutputValueClass(IntWritable.class);
	    job.setOutputKeyClass(IntWritable.class);
	    job.setOutputValueClass(IntWritable.class);
	    job.setPriorityClass(IntWritable.class);

	    job.setNumMapTasks(partitions);
	    job.setNumReduceTasks(partitions);
	    
	    JobClient.runJob(job);
	    return 0;
	}
	
	static int printUsage() {
		System.out.println("conncomp [-p <partitions>] [-k <options>] [-qexlen <ex queue len>] " +
				"[-i <snapshot interval>] [-t <termination time>] input output");
		ToolRunner.printGenericCommandUsage(System.out);
		return -1;
	}
	
	@Override
	public int run(String[] args) throws Exception {
	    List<String> other_args = new ArrayList<String>();
	    for(int i=0; i < args.length; ++i) {
	      try {
	        if ("-p".equals(args[i])) {
	        	partitions = Integer.parseInt(args[++i]);
	        } else if ("-k".equals(args[i])) {
	        	topk = Integer.parseInt(args[++i]);
	        } else if ("-q".equals(args[i])) {
	        	exetop = Integer.parseInt(args[++i]);
	        } else if ("-i".equals(args[i])) {
	        	snapinterval = Long.parseLong(args[++i]);
	        } else if ("-t".equals(args[i])) {
	        	stoptime = Long.parseLong(args[++i]);
	        } else {
	          other_args.add(args[i]);
	        }
	      } catch (NumberFormatException except) {
	        System.out.println("ERROR: Integer expected instead of " + args[i]);
	        return printUsage();
	      } catch (ArrayIndexOutOfBoundsException except) {
	        System.out.println("ERROR: Required parameter missing from " +
	                           args[i-1]);
	        return printUsage();
	      }
	    }
	    // Make sure there are exactly 2 parameters left.
	    if (other_args.size() != 2) {
	      System.out.println("ERROR: Wrong number of parameters: " +
	                         other_args.size() + " instead of 2.");
	      return printUsage();
	    }
	    
	    input = other_args.get(0);
	    output = other_args.get(1);
	    subGraphDir = input + "_subgraph";
	    
	    Distributor.partition(input, subGraphDir, partitions, IntWritable.class, HashPartitioner.class);
	    conncomp();
	    
		return 0;
	}

	/**
	 * @param args
	 * @throws Exception 
	 */
	public static void main(String[] args) throws Exception {
		// TODO Auto-generated method stub
		int res = ToolRunner.run(new Configuration(), new ConnectComponent(), args);
	    System.exit(res);
	}
}
```