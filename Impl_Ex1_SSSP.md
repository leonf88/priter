# Introduction #
SSSP problem is a classical problem that derives the shortest distance from a source node to other nodes in a graph. Under a parallel computing environment, the most straightforward approach to find the shortest distance is to use breadth first search (BFS).

In the MapReduce model, each iteration performs the above computation for all nodes. As a result, iteration _i_ computes the shortest distance up to _i_ hops away. However, as we can see from Dijsktra's algorithm, it is more efficient if we expand the node with the shortest distance first since it is more likely that node has already determine its shortest distance. Therefore, a prioritized execution of the iterative process is desirable.

A prioritized execution of the SSSP algorithm can be described as follows. In each iteration, only a subset of nodes perform the iterative computation. The selection of the subset is according to a priority. First, a node is eligible to perform the iterative computation only if it has obtained a shorter distance since its last time the iterative computation is performed on the node. Second, among all eligible nodes, the nodes with shorter distance have a higher priority to be activated to send its distance to its neighbors. Third, when the iterative operation is performed on a node, the node gets the distance values from neighboring nodes only if the neighboring nodes were activated since the node was last activated. The operation derives the shortest distance for the node among all distance values received.

# Details #

**_Activator_**
```
public class SSSPActivator extends PrIterBase implements 
	Activator<IntWritable, FloatWritable, FloatWritable> {
	
	private String subGraphsDir;
	private int partitions;
	private int startnode;
	private int kvs = 0;
	private int iter = 0;
	
	//graph in local memory
	private HashMap<Integer, ArrayList<Link>> linkList = new HashMap<Integer, ArrayList<Link>>();

	private class Link{
		int node;
		float weight;
		
		public Link(int n, float w){
			node = n;
			weight = w;
		}
		
		@Override
		public String toString() {
			return new String(node + "\t" + weight);
		}
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
					ArrayList<Link> links = new ArrayList<Link>();
					StringTokenizer st = new StringTokenizer(linkstring);
					while(st.hasMoreTokens()){
						String link = st.nextToken();
						//System.out.println(link);
						String item[] = link.split(",");
						Link l = new Link(Integer.parseInt(item[0]), Float.parseFloat(item[1]));
						links.add(l);
					}

					this.linkList.put(Integer.parseInt(node), links);
				}
			}
			reader.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	
	@Override
	public void configure(JobConf job){   
	    startnode = job.getInt(MainDriver.START_NODE, 0);
	    int taskid = Util.getTaskId(job);
	    partitions = job.getInt("priter.graph.partitions", -1);
		loadGraphToMem(job, taskid);
	}
	
	@Override
	public void activate(IntWritable key, FloatWritable value,
			OutputCollector<IntWritable, FloatWritable> output, Reporter report)
			throws IOException {
		kvs++;
		report.setStatus(String.valueOf(kvs));

		float distance = value.get();
		if(distance != Integer.MAX_VALUE){	
			int node = key.get();
			ArrayList<Link> links = null;
			links = this.linkList.get(node);
			
			if(links == null) {
				System.out.println("no links for node " + node);
				for(int i=0; i<partitions; i++){
					output.collect(new IntWritable(i), new FloatWritable(Float.MAX_VALUE));
				}
				return;
			}
				
			for(Link l : links){				
				output.collect(new IntWritable(l.node), new FloatWritable(distance + l.weight));
			}
		} else{
			for(int i=0; i<partitions; i++){
				output.collect(new IntWritable(i), new FloatWritable(Float.MAX_VALUE));
			}			
		}
	}

	@Override
	public void initStarter(InputPKVBuffer<IntWritable, FloatWritable> starter)
			throws IOException {
		starter.init(new IntWritable(startnode), new FloatWritable(0));
	}

	@Override
	public void iterate() {
		System.out.println((iter++) + " passes " + kvs + " activations");
	}
}

```

**_Updator_**
```
public class SSSPUpdater extends PrIterBase implements
		Updater<IntWritable, FloatWritable, FloatWritable> {
	private int workload = 0;
	private int iterate = 0;
	
	@Override
	public void iterate() {
		iterate++;
		System.out.println("iteration " + iterate + " total parsed " + workload);
	}
	
	@Override
	public FloatWritable resetiState() {
		return new FloatWritable(Float.MAX_VALUE);
	}
	

	@Override
	public void initStateTable(
			OutputPKVBuffer<IntWritable, FloatWritable, FloatWritable> stateTable) {
	}

	@Override
	public FloatWritable decidePriority(IntWritable key, FloatWritable iState) {
		return new FloatWritable(-iState.get());
	}

	@Override
	public FloatWritable decideTopK(IntWritable key, FloatWritable cState) {
		return new FloatWritable(-cState.get());
	}

	@Override
	public void updateState(IntWritable key, Iterator<FloatWritable> values,
			OutputPKVBuffer<IntWritable, FloatWritable, FloatWritable> buffer, Reporter report)
			throws IOException {
		workload++;	
		report.setStatus(String.valueOf(workload));
		
		float min_len = values.next().get();
		synchronized(buffer.stateTable){
			PriorityRecord<FloatWritable, FloatWritable> pkvRecord;	
			if(buffer.stateTable.containsKey(key)){
				pkvRecord = buffer.stateTable.get(key);
				float cState = pkvRecord.getcState().get();
				if(min_len < cState){
					buffer.stateTable.get(key).getiState().set(min_len);
					buffer.stateTable.get(key).getcState().set(min_len);
					buffer.stateTable.get(key).getPriority().set(-min_len);
				}
			}else{
				pkvRecord = new PriorityRecord<FloatWritable, FloatWritable>(
						new FloatWritable(-min_len), new FloatWritable(min_len), new FloatWritable(min_len));
				buffer.stateTable.put(new IntWritable(key.get()), pkvRecord);
			}
		}
	}
}

```

**_Main_**
```
public class SSSP extends Configured implements Tool {
	private String input;
	private String output;
	private String subGraphDir;
	private int partitions;
	private int topk;
	private int startnode;
	private int queuelen;
	private long snapinterval = 5000;
	private float stopthresh;
	
	private int sssp() throws IOException{
	    JobConf job = new JobConf(getConf());
	    String jobname = "shortest path";
	    job.setJobName(jobname);
       
	    job.set(MainDriver.SUBGRAPH_DIR, subGraphDir);
	    job.setInt(MainDriver.START_NODE, startnode);
	    if(partitions == 0) partitions = Util.getTTNum(job);			//set default partitions = num of task trackers
	    
	    FileInputFormat.addInputPath(job, new Path(input));
	    FileOutputFormat.setOutputPath(job, new Path(output));
	    job.setOutputFormat(TextOutputFormat.class);
	    
	    //set for iterative process   
	    job.setBoolean("priter.job", true);
	    job.setInt("priter.graph.partitions", partitions);				//graph partitions
	    job.setLong("priter.snapshot.interval", snapinterval);			//snapshot interval	 
	    job.setInt("priter.snapshot.topk", topk);						//topk 
	    job.setInt("priter.queue.length", queuelen);						//execution queue
	    job.setFloat("priter.stop.difference", stopthresh);				//termination check
	    
	    job.setJarByClass(SSSP.class);
	    job.setActivatorClass(SSSPActivator.class);	
	    job.setUpdaterClass(SSSPUpdater.class);
	    job.setMapOutputKeyClass(IntWritable.class);
	    job.setMapOutputValueClass(FloatWritable.class);
	    job.setOutputKeyClass(IntWritable.class);
	    job.setOutputValueClass(FloatWritable.class);
	    job.setPriorityClass(FloatWritable.class);    

	    job.setNumMapTasks(partitions);
	    job.setNumReduceTasks(partitions);
	    
	    JobClient.runJob(job);
	    return 0;
	}
	
	static int printUsage() {
		System.out.println("sssp [-p <partitions>] [-k <options>] [-qlen <qportion>] [-s <source node>]" +
				"[-i <snapshot interval>] [-t <termination threshod>] input output");
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
	        } else if ("-qlen".equals(args[i])) {
	        	queuelen = Integer.parseInt(args[++i]);
	        } else if ("-s".equals(args[i])) {
	        	startnode = Integer.parseInt(args[++i]);
	        } else if ("-i".equals(args[i])) {
	        	snapinterval = Long.parseLong(args[++i]);
	        } else if ("-t".equals(args[i])) {
	        	stopthresh = Float.parseFloat(args[++i]);
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
	    sssp();
	    
		return 0;
	}

	/**
	 * @param args
	 * @throws Exception 
	 */
	public static void main(String[] args) throws Exception {
		// TODO Auto-generated method stub
		int res = ToolRunner.run(new Configuration(), new SSSP(), args);
	    System.exit(res);
	}

}
```