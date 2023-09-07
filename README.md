# JVector 
JVector is a pure Java, zero dependency, embedded vector search engine, used by DataStax Astra DB and Apache Cassandra.

What is JVector?
- Algorithmic-fast. JVector uses dstate of the art graph algorithms inspired by DiskANN and related research that offer high recall and low latency.
- Implementation-fast. JVector uses the Panama SIMD API to accelerate index build and queries.
- Memory efficient. JVector compresses vectors using product quantization so they can stay in memory during searches.  (As part of our PQ implementation, our SIMD-accelerated kmeans implementation is 3x faster than Apache Commons Math.)
- Disk-aware. JVector’s disk layout is designed to do the minimum necessary iops at query time.
- Concurrent.  Index builds scale linearly to at least 32 threads.  Double the threads, half the build time.
- Incremental. Query your index as you build it.  No delay between adding a vector and being able to find it in search results.
- Easy to embed. API designed for easy embedding, by people using it in production.

## JVector basics
- Add io.github.jbellis / jvector as a dependency.
- GraphIndexBuilder is the entry point for building a graph.  You will need to implement 
  RandomAccessVectorValues to provide vectors to the builder;  
  ListRandomAccessVectorValues is a good starting point.  If all your vectors
  are in the provider
  up front, you can just call build() and it will parallelize the build across
  all available cores.  Otherwise you can call addGraphNode as you add vectors; 
  this is non-blocking and can be called concurrently from multiple threads.
  Call GraphIndexBuilder.complete when you are done adding vectors.  This will
  optimize the index and make it ready to write to disk.  (Graphs that are
  in the process of being built can be searched at any time; you do not have to call
  complete first.)
- JVector represents vectors in the index as the ordinal (int) corresponding to their
  index in the RandomAccessVectorValues you provided.  You can get the original vector
  back with GraphIndex.getVector, if necessary, but since this is a disk-backed index
  you should design your application to avoid doing so if possible.
- GraphSearcher is the entry point for searching.  Results come back as a SearchResult object that contains node IDs and scores, in 
  descending order of similarity to the query vector.  GraphSearcher objects are re-usable,
  so unless you have a very simple use case you should use GraphSearcher.Builder to
  create them; GraphSearcher::search is also available with simple defaults, but calling it
  will instantiate a new GraphSearcher every time so performance will be worse.
## DiskANN and Product Quantization 
JVector implements [DiskANN](https://suhasjs.github.io/files/diskann_neurips19.pdf)-style 
search, meaning that vectors can be compressed using product quantization so that searches
can be performed using the compressed representation that is kept in memory.  You can enable
this with the following steps:
- Create a ProductQuantization object with your vectors.  This will take some time
  to compute the codebooks.
- Use ProductQuantization.encode or encodeAll to encode your vectors.  
- Create a CompressedVectors object from the encoded vectors.
- Create a NeighborSimilarity.ApproximateScoreFunction for your query that uses the
  ProductQuantization object and CompressedVectors to compute scores, and pass this
  to the GraphSearcher.search method.

## Saving and loading indexes

- OnDiskGraphIndex and CompressedVectors have write() methods to save state to disk.
  They initialize from disk using their constructor and load() methods, respectively.
  Writing just requires a DataOutput, but reading requires an 
  implementation of RandomAccessReader to wrap your
  preferred i/o class for best performance. 
- Building a graph does not technically require your RandomAccessVectorValues object
  to live in memory, but it will perform much better if it does.  OnDiskGraphIndex,
  by contrast, is designed to live on disk and use minimal memory otherwise.
- You can optionally wrap OnDiskGraphIndex in a CachingGraphIndex to keep the most commonly accessed
  nodes in memory.

## Sample code
  
- The SiftSmall class demonstrates how to put all of the above together to index and search the
  "small" SIFT dataset of 10,000 vectors.
- The Bench class performs grid search across the GraphIndexBuilder parameter space to find
  the best tradeoffs between recall and throughput.  You can use plot_output.py to graph the pareto-optimal
  points found by Bench.

# Developing and Testing

You can run SiftSmall and Bench directly to get an idea of what all is going on here. Bench 
requires some datasets to be downloaded from https://github.com/erikbern/ann-benchmarks. The files used by SiftSmall can be found in the siftsmall directory in the project root. 

To run either class, you can use the Maven exec-plugin via the following incantations:
```mvn clean install exec:exec@bench``` 
or for Sift:
```mvn clean install exec:exec@sift```

To compile for a specific JDK, you can use the targeted execution defined in the pom:
- `mvn clean compiler:compile@jdk11` for JDK 11
- `mvn clean compiler:compile@jdk20` for JDK 20

Similar to the compile executions, a JAR file can be generated via:
- `mvn jar:jar@jar-jdk11` for JDK 11
- `mvn jar:jar@jar-jdk20` for JDK 20

In both cases, you must have invoked the compile target for that specific JDK, or the resulting jar file will be empty.  