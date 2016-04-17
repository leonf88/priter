# Introduction #

```
template <class K>
struct Partitioner {
    virtual int operator()(const K& k, int partitons) = 0;
};

template <class K, class V>
struct Initializer{
    virtual void initialize(const K& k, V* msg, V* state) = 0;
};

template <class V>
struct Accumulator {
    virtual void accum-msg(V* msg, const V& in_msg) = 0;
};

template <class K, class V, class L>
struct Updator{
    virtual void update(const K& k, const V& msg, V* state, const vector<L>& adj) = 0;
    virtual void reset(const K& k, V* msg) = 0;
};

template <class K, class V>
struct Scheduler {
    double schedule_portion;
    virtual V priority(const K& k, const V& msg) = 0;
};

template <class K, class V>
struct TermChecker {
    int termcheck_interval;  
    virtual V local_status(Iterator<K, V>* statetable) = 0;
    virtual bool terminate(vector<V> local_statuses) = 0;
};




struct PagerankInitializer : public Initializer<int, float> {
    void initialize(const int& k, float* msg, float* state){
        *msg = 0.2;
        *state = 0;
    }
}

struct PagerankUpdator : public Updator<int, float, int> {
    void update(const int& k, const float& msg, float* state, const vector<int>& adj){
        *state = msg;
        int size = (int) adj.size();
	vector<int>::iterator it;
	for(it=adj.begin(); it!=adj.end(); it++){
            int target = *it;
            send(target, msg * 0.8 / size);
	}
    }
    void reset(const int& k, float* msg){
        *msg = 0;
    }
}

struct PagerankScheduler : public Scheduler<int, float> {
    schedule_portion = 0.01;
    float priority(const int& k, const float& msg){
        return msg;
    }
}

struct PagerankTermChecker : public TermChecker<int, float> {
    float local_status(Iterator<int, float>* statetable){
        float partial_curr = 0;
        while(!statetable->done()){
            statetable->Next();
            partial_curr += statetable->state();
        }
        return partial_curr;
    }
    
    bool terminate(vector<float> local_statuses){
        vector<float>::iterator it;
        for(it=partials.begin(); it!=partials.end(); it++){
                float partial = *it;
                curr += partial;
        }
        
        if(curr > 99999999){
            return true;
        }else{
            return false;
        }
    }
}
```