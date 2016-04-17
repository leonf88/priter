# Introduction #
Adsorption is a graph-based label propagation algorithm, which provides personalized recommendation for contents (e.g., video, music, document, product). The concept of _label_ indicates a certain common feature of the entities. Adsorption's label propagation proceeds to label all of the nodes based on the graph structure, ultimately producing a probability distribution over labels for each node.

Given a weighted graph, each node carries a probability distribution on the label, and it is initially assigned with an _initial distribution_. The algorithm proceeds as follows. For each
node, it iteratively computes the weighted average of label distributions from its neighboring nodes, and then uses the random walk probabilities to estimate a new label distribution for a node.


# Details #

**_Activator_**
```
public class AdsorptionActivator extends PrIterBase implements
		Activator<IntWritable, DoubleWritable, DoubleWritable> {

	private String subGraphsDir;
	private int kvs = 0;
	private int iter = 0;
	private int partitions;
	
	class LinkWeight {
		public int link;
		public double weight;
		
		public LinkWeight(int link, double weight){
			this.link = link;
			this.weight = weight;
		}
	}
	//graph in local memory
	private HashMap<Integer, ArrayList<LinkWeight>> linkList = new HashMap<Integer, ArrayList<LinkWeight>>();
	 
	private synchronized void loadGraphToMem(JobConf conf, int n) {
		subGraphsDir = conf.get(MainDriver.SUBGRAPH_DIR);
		Path remote_link = new Path(subGraphsDir + "/part" + n);
		
		FileSystem hdfs = null;
		try{
			hdfs = FileSystem.get(conf);
			FSDataInputStream in = hdfs.open(remote_link);
			BufferedReader reader = new BufferedReader(new InputStreamReader(in));
			
			String line;
			while((line = reader.readLine()) != null){
				int index = line.indexOf("\t");
				if(index != -1){
					String node = line.substring(0, index);
					
					String linkstring = line.substring(index+1);
					ArrayList<LinkWeight> links = new ArrayList<LinkWeight>();
					StringTokenizer st = new StringTokenizer(linkstring);
					while(st.hasMoreTokens()){
						String link = st.nextToken();
						//System.out.println(link);
						String item[] = link.split(",");
						LinkWeight l = new LinkWeight(Integer.parseInt(item[0]), Double.parseDouble(item[1]));
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
	public void configure(JobConf job) {
		int taskid = Util.getTaskId(job);
		this.loadGraphToMem(job, taskid);
		partitions = job.getInt("priter.graph.partitions", 1);
	}
	
	@Override
	public void initStarter(InputPKVBuffer<IntWritable, DoubleWritable> starter)
			throws IOException {
		double initvalue = Adsorption.RETAINFAC * 10000;
		int count = 0;
		for(int k : linkList.keySet()){
			if(++count > 25) break;
			starter.init(new IntWritable(k), new DoubleWritable(initvalue));
		}
	}

	@Override
	public void activate(IntWritable key, DoubleWritable value,
			OutputCollector<IntWritable, DoubleWritable> output, Reporter report)
			throws IOException {
			
		kvs++;
		report.setStatus(String.valueOf(kvs));
		
		int page = key.get();
		ArrayList<LinkWeight> links = null;
		links = this.linkList.get(key.get());

		if(links == null){
			System.out.println("no links found for page " + page);
			for(int i=0; i<partitions; i++){
				output.collect(new IntWritable(i), new DoubleWritable(0));
			}
			return;
		}	
		double delta = value.get() * Adsorption.DAMPINGFAC;
		
		for(LinkWeight link : links){
			output.collect(new IntWritable(link.link), new DoubleWritable(delta*link.weight));
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
public class AdsorptionUpdater extends PrIterBase implements
		Updater<IntWritable, DoubleWritable, DoubleWritable> {
	
	private JobConf job;
	private int workload = 0;
	private int iterate = 0;
	private HashMap<Integer, Double> weightMap = new HashMap<Integer, Double>();

	@Override
	public void configure(JobConf job) {		
		this.job = job;
	}
	
	@Override
	public void iterate() {
		iterate++;
		System.out.println("iteration " + iterate + " total parsed " + workload);
	}

	@Override
	public void initStateTable(
			OutputPKVBuffer<IntWritable, DoubleWritable, DoubleWritable> stateTable) {
		String subGraphsDir = job.get(MainDriver.SUBGRAPH_DIR);
		int taskid = Util.getTaskId(job);
		Path remote_link = new Path(subGraphsDir + "/part" + taskid);
		double initvalue = Adsorption.RETAINFAC * 10000;
		
		FileSystem hdfs = null;
		try{
			hdfs = FileSystem.get(job);
			FSDataInputStream in = hdfs.open(remote_link);
			BufferedReader reader = new BufferedReader(new InputStreamReader(in));
			
			String line;
			while((line = reader.readLine()) != null){
				int index = line.indexOf("\t");
				if(index != -1){
					int node = Integer.parseInt(line.substring(0, index));
					
					double weight = 0.0;
					String linkstring = line.substring(index+1);
					StringTokenizer st = new StringTokenizer(linkstring);
					while(st.hasMoreTokens()){
						String link = st.nextToken();
						String field[] = link.split(",");
						weight += Double.parseDouble(field[1]);
					}
					this.weightMap.put(node, weight);	
					stateTable.init(new IntWritable(node), new DoubleWritable(0), new DoubleWritable(initvalue));
				}
			}
			reader.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	@Override
	public DoubleWritable resetiState() {
		return new DoubleWritable(0.0);
	}

	@Override
	public DoubleWritable decidePriority(IntWritable key, DoubleWritable iState) {
		Double weight = weightMap.get(key.get());
		if(weight == null){
			System.out.println(key + "\tnull");
			return new DoubleWritable(0.0);
		}else{
			return new DoubleWritable(iState.get() * weightMap.get(key.get()));
		}	
	}
	
	@Override
	public DoubleWritable decideTopK(IntWritable key, DoubleWritable cState) {
		return new DoubleWritable(cState.get());	
	}

	@Override
	public void updateState(IntWritable key, Iterator<DoubleWritable> values,
			OutputPKVBuffer<IntWritable, DoubleWritable, DoubleWritable> buffer, Reporter report)
			throws IOException {
		workload++;		
		report.setStatus(String.valueOf(workload));
		
		double delta = 0.0;
		while(values.hasNext()){				
			delta += values.next().get();	
		}
		
		synchronized(buffer.stateTable){
			PriorityRecord<DoubleWritable, DoubleWritable> pkvRecord;	
			if(buffer.stateTable.containsKey(key)){
				pkvRecord = buffer.stateTable.get(key);
	
				double iState = pkvRecord.getiState().get() + delta;
				double cState = pkvRecord.getcState().get() + delta;
				buffer.stateTable.get(key).getiState().set(iState);
				buffer.stateTable.get(key).getcState().set(cState);
				buffer.stateTable.get(key).getPriority().set(iState);
			}else{
				pkvRecord = new PriorityRecord<DoubleWritable, DoubleWritable>(
						new DoubleWritable(delta), new DoubleWritable(delta), new DoubleWritable(delta));
				buffer.stateTable.put(new IntWritable(key.get()), pkvRecord);
			}
		}
	}
}

```

**_Main_**
```
public class Adsorption extends Configured implements Tool {
	private String input;
	private String output;
	private String subGraphDir;
	private int partitions;
	private int topk;
	private float qportion;
	private long snapinterval = 20000;
	private float stopthresh;
	
	//damping factor
	public static final double DAMPINGFAC = 0.8;
	public static final double RETAINFAC = 0.2;

	
	private int adsorption() throws IOException{
	    JobConf job = new JobConf(getConf());
	    String jobname = "adsorption";
	    job.setJobName(jobname);
       
	    job.set(MainDriver.SUBGRAPH_DIR, subGraphDir);
	    if(partitions == 0) partitions = Util.getTTNum(job);			//set default partitions = num of task trackers
	    
	    FileInputFormat.addInputPath(job, new Path(input));
	    FileOutputFormat.setOutputPath(job, new Path(output));
	    job.setOutputFormat(TextOutputFormat.class);
	    
	    //set for iterative process   
	    job.setBoolean("priter.job", true);
	    job.setInt("priter.graph.partitions", partitions);				//graph partitions
	    job.setLong("priter.snapshot.interval", snapinterval);			//snapshot interval	
	    job.setInt("priter.snapshot.topk", topk);						//topk
	    job.setFloat("priter.queue.portion", qportion);					//activation queue
	    job.setFloat("priter.stop.difference", stopthresh);				//termination check
	    
	        
	    job.setJarByClass(Adsorption.class);
	    job.setActivatorClass(AdsorptionActivator.class);	
	    job.setUpdaterClass(AdsorptionUpdater.class);
	    job.setMapOutputKeyClass(IntWritable.class);
	    job.setMapOutputValueClass(DoubleWritable.class);
	    job.setOutputKeyClass(IntWritable.class);
	    job.setOutputValueClass(DoubleWritable.class);
	    job.setPriorityClass(DoubleWritable.class);

	    job.setNumMapTasks(partitions);
	    job.setNumReduceTasks(partitions);
	    
	    JobClient.runJob(job);
	    return 0;
	}
	
	static int printUsage() {
		System.out.println("adsorption [-p <partitions>] [-k <options>] [-q <qportion>] " +
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
	        } else if ("-q".equals(args[i])) {
	        	qportion = Float.parseFloat(args[++i]);
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
	    adsorption();
	    
		return 0;
	}

	/**
	 * @param args
	 * @throws Exception 
	 */
	public static void main(String[] args) throws Exception {
		int res = ToolRunner.run(new Configuration(), new Adsorption(), args);
	    System.exit(res);
	}
}
```