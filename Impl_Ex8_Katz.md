# Introduction #

Katz metric computation.


# Details #
**_Activator_**
```
public class KatzActivator extends PrIterBase implements
	Activator<IntWritable, FloatWritable, FloatWritable> {

	private String subGraphsDir;
	private int kvs = 0;
	private int iter = 0;
	private int partitions;
	private float beta;
	
	//graph in local memory
	private HashMap<Integer, ArrayList<Integer>> linkList = new HashMap<Integer, ArrayList<Integer>>();
	 
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
			reader.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

	@Override
	public void configure(JobConf job) {
		int taskid = Util.getTaskId(job);
		partitions = job.getInt("priter.graph.partitions", 1);
		beta = job.getFloat(MainDriver.KATZ_BETA, (float)0.05);
		loadGraphToMem(job, taskid);
	}
	
	@Override
	public void initStarter(InputPKVBuffer<IntWritable, FloatWritable> starter)
			throws IOException {	
		starter.init(new IntWritable(0), new FloatWritable(1000000));
	}

	@Override
	public void activate(IntWritable key, FloatWritable value,
			OutputCollector<IntWritable, FloatWritable> output, Reporter report)
			throws IOException {
		kvs++;
		report.setStatus(String.valueOf(kvs));
		
		int page = key.get();
		ArrayList<Integer> links = null;
		links = this.linkList.get(key.get());

		if(links == null){
			System.out.println("no links found for node " + page);
			for(int i=0; i<partitions; i++){
				output.collect(new IntWritable(i), new FloatWritable(0));
			}
			return;
		}	
		float delta = value.get() * beta;
		
		for(int link : links){
			output.collect(new IntWritable(link), new FloatWritable(delta));
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

public class KatzUpdater extends PrIterBase implements
		Updater<IntWritable, FloatWritable, FloatWritable> {
	
	private int workload = 0;
	private int iterate = 0;
	private String subGraphsDir;
	private HashMap<Integer, Integer> degreeMap = new HashMap<Integer, Integer>();

	private synchronized void readDegree(JobConf conf, int n){
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
					String[] links = linkstring.split(" ");
					int degree = links.length;
					this.degreeMap.put(Integer.parseInt(node), degree);
				}	
			}
			reader.close();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	
	@Override
	public void configure(JobConf job) {	
		readDegree(job, Util.getTaskId(job));
	}

	@Override
	public void iterate() {
		iterate++;
		System.out.println("iteration " + iterate + " total parsed " + workload);
	}

	@Override
	public void initStateTable(OutputPKVBuffer<IntWritable, FloatWritable, FloatWritable> stateTable) {}

	@Override
	public FloatWritable resetiState() {
		return new FloatWritable(0);
	}
	
	@Override
	public FloatWritable decidePriority(IntWritable key, FloatWritable iState) {
		if(this.degreeMap.get(key.get()) != null){
			return new FloatWritable(iState.get()*this.degreeMap.get(key.get()));
		}else{
			return new FloatWritable(iState.get());
		}
	}
	
	@Override
	public FloatWritable decideTopK(IntWritable key, FloatWritable cState) {
		return new FloatWritable(cState.get());
	}

	@Override
	public void updateState(IntWritable key, Iterator<FloatWritable> values,
			OutputPKVBuffer<IntWritable, FloatWritable, FloatWritable> buffer, Reporter report)
			throws IOException {
		workload++;		
		report.setStatus(String.valueOf(workload));
		
		float delta = 0;
		while(values.hasNext()){				
			delta += values.next().get();	
		}

		synchronized(buffer.stateTable){
			PriorityRecord<FloatWritable, FloatWritable> pkvRecord;	
			if(buffer.stateTable.containsKey(key)){
				pkvRecord = buffer.stateTable.get(key);
	
				float iState = pkvRecord.getiState().get() + delta;
				float cState = pkvRecord.getcState().get() + delta;
				buffer.stateTable.get(key).getiState().set(iState);
				buffer.stateTable.get(key).getcState().set(cState);
				buffer.stateTable.get(key).getPriority().set(iState*this.degreeMap.get(key.get()));
			}else{
				pkvRecord = new PriorityRecord<FloatWritable, FloatWritable>(
						new FloatWritable(delta), new FloatWritable(delta), new FloatWritable(delta));
				buffer.stateTable.put(new IntWritable(key.get()), pkvRecord);
			}
		}
	}
}
```

**_Main_**
```

public class Katz extends Configured implements Tool {
	private String input;
	private String output;
	private String subGraphDir;
	private int partitions;
	private int topk;
	private float qportion;
	private long snapinterval = 10000;
	private float stopthresh;
	private float beta;

	private int katz() throws IOException{
	    JobConf job = new JobConf(getConf());
	    String jobname = "katz metric";
	    job.setJobName(jobname);
       
	    job.set(MainDriver.SUBGRAPH_DIR, subGraphDir);
	    job.setFloat(MainDriver.KATZ_BETA, beta);
	    if(partitions == 0) partitions = Util.getTTNum(job);			//set default partitions = num of task trackers
	    
	    FileInputFormat.addInputPath(job, new Path(input));
	    FileOutputFormat.setOutputPath(job, new Path(output));
	    job.setOutputFormat(TextOutputFormat.class);
	    	    
	    //set for iterative process   
	    job.setBoolean("priter.job", true);
	    job.setInt("priter.graph.partitions", partitions);				//graph partitions
	    job.setLong("priter.snapshot.interval", snapinterval);			//snapshot interval	
	    job.setInt("priter.snapshot.topk", topk);						//topk
	    job.setFloat("priter.queue.portion", qportion);					//execution queue
	    job.setFloat("priter.stop.difference", stopthresh);				//termination check
	    		
	    job.setJarByClass(Katz.class);
	    job.setActivatorClass(KatzActivator.class);	
	    job.setUpdaterClass(KatzUpdater.class);
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
		System.out.println("katz [-p <partitions>] [-k <options>] [-q <qportion>] [-beta <beta>]" +
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
	        } else if ("-beta".equals(args[i])) {
	        	beta = Float.parseFloat(args[++i]);
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
	    katz();
	    
		return 0;
	}

	/**
	 * @param args
	 * @throws Exception 
	 */
	public static void main(String[] args) throws Exception {
		// TODO Auto-generated method stub
		int res = ToolRunner.run(new Configuration(), new Katz(), args);
	    System.exit(res);
	}
}
```