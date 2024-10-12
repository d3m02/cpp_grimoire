> [CppCon 2014: Chandler Carruth "Efficiency with Algorithms, Performance with Data Structures"](https://youtu.be/fHNmRkzxHWs?si=NPGNxepBjvTzL1jr])


High level thesis:
* Efficiency through Algorithms
	* Efficiency: how much work is required by a task. Improving efficiency involves doing less work. An efficient program is one which does the minimum amount of work to accomplish a given task (in absolute, meaning that there nothing left for reducing amount of operations to do) 
* Performance through Data Structures. 
	* Performance - how quickly program does it's work. Improving performance involves doing same amount of work faster in time. But there is no such thing as "performant", there is essentially no point at which a program cannot to work any faster (until you hit [Bremermann's limit](https://en.wikipedia.org/wiki/Bremermann%27s_limit)). 
	* Improving performance mostly mean involving in computation all hardware available resources, lighting up as much as possible transistors. Every circuit not used in computer is wasting power. But worth mentioning that using more parts of PC without improving performance is waste of power (watts). 

Improving algorithmic efficiency requires finding a different way of solving the problem. 

Let's take for example sub-string searching.
Initially, we have $O(n^{2})$ algorithm (brute force);
Next, we have [Knuth-Morris-Pratt](https://www.geeksforgeeks.org/kmp-algorithm-for-pattern-searching/) (a table to skip;
Finally, we have [Boyer-Moore](https://www.geeksforgeeks.org/boyer-moore-algorithm-for-pattern-searching/) (use the end of the needle);

Next perspective is do less work by not wasting effort. 
For example 
```c++
std::vector<X> someSillyFunction(int amount)
{
	std::vector<X> result;
	for (int i = 0; i < n; ++i)
		result.push_back(X(...));
	return result; 
}
```
simple `result.reserve(n)` can make function a little bit complex, but can't help gain some efficiency by avoiding redundant memory allocation (and also makes code a little bit more clear).

Level 2: 
```c++
X* getX(std::string key, 
	    std::unordered_map<std::string, std::unique_ptr<X>>& cache)
{
	if (cache[key])
		return cache[key].get();
	
	cache[key] = std::make_unique<X>(...);
	return cache[key].get();
}
```
This code is linear (to the length of key?). Problem here that code every time makes caching key (unordered_map work takes key, calculate cache, looks hash into hash table, taking pointers, etc). And here we do it 4 times and get exact same result 
```c++
X* getX(std::string key, 
	    std::unordered_map<std::string, std::unique_ptr<X>>& cache)
{
	std::unique_ptr<X>& entry = cache[key];
	
	if (entry)
		return entry.get();
	
	entry = std::make_unique<X>(...);
	return entry.get();
}
```

Data structures: 
"Discontinuous data structures are the root of all (performance) evil".
Lined lists - harmful for performance🤨
	Modern CPU are too fast (note, 2014 video and speaker takes for reference Intel Sandy Bridge): 1kkk cycles per second, 12+ cores, 3+ executions ports per core (how many instruction can be executed simultaneously) = 36kkk instructions per second: around 50% of time spent waiting for data.

We need faster memory

| One cycle on a 3 Ghz CPU           | 1           | ns  |         |
| ---------------------------------- | ----------- | --- | ------- |
| L1 cache reference                 | 0.5         | ns  |         |
| Branch mispredict                  | 5           | ns  |         |
| L2 cache reference                 | 7           | ns  |         |
| Mutex lock/unlock                  | 25          | ns  |         |
| Main memory Reference              | 100         | ns  |         |
| Compress 1K bytes with Snappy      | 3,000       | ns  |         |
| Send 1k bytes over 1 Gbps net      | 10,000      | ns  | 0.01 ms |
| Read 4k randomly from SSD          | 150,000     | ns  | 0.15 ms |
| Read 1 MB sequentially from memory | 250,000     | ns  | 0.25 ms |
| Round trip within same datacenter  | 500,000     | ns  | 0.5 ms  |
| Read 1 MB sequentially from SSD    | 1,000,000   | ns  | 1 ms    |
| Disk seek                          | 10,000,000  | ns  | 10 ms   |
| Read 1 MB sequentially from disk   | 20,000,000  | ns  | 20 ms   |
| Send packet CA->netherlands->CA    | 150,000,000 | ns  | 150 ms  |
CPUs have a hierarchical cache system which work in such way, that first CPU search data in L1, small but extremely fast memory, if not - search in L2, => L3 => main memory => etc.  
So everything mostly designed to have contiguous data and small amount of that data to fit as higher as possible in that hierarchy. 

For example, std::list
* Doubly-linked list
* Each node <u>separately</u> allocated  
* All traversal operations chase pointers to totally new memory (which in most cases fall down in cache hierarchy)
* In most cases, every step in a cache miss 
* Only use this when you rarely traverse the list, but very frequently update the list. For example it can be useful for algorithmically improvement of updating data structure. When traverse in same order of magnitude number of times to update data of structure - lists worth choice. 

Stacks, queues?
* just use std::vector (although, it's already a perfectly good stack)
* if the queue can have a total upper bound and/or is short-lived, vector also can be used as queue with and index into the front
	* grow the vector forever, chase the tail with the index 
	* Insert items into the back; don't remove from the front; keep an index moving and since we don't remove from the front - it doesn't invalidate index (but push_back invalidate iterators) - invalidating less often. 

std::map sometimes even worse than std::list. 
* std::map is just a linked list, oriented as a binary tree
* all the downsides of lined lists 
* insertion and removal are also partial traversals 
* even with hinting, every rebalancing is a traversal 

std::unordered_map 
* Essentially required to be implemented with buckets of key-value for each key entry
* buckets = linked lists (we can have bucket iterators, iterate the element of the bucked and observer begin() and end()) 
* always have some pointer chasing. 


Data structures and algorithms unfortunately most of time don't work as "pick best algorithm, pick best data structure", they are tightly coupled. Both factors should be keep in mind to balance between them. Algorithms can also influence the data access pattern regardless of the data structure used. 