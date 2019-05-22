# **一、 硬件环境选择**

如果有条件，尽可能使用SSD硬盘， 不错的CPU。ES的厉害之处在于ES本身的分布式架构以及lucene的特性；IO的提升，会极大改进ES的速度和性能；内存配置方面，一般来说，64G内存的机器节点较佳。

# **二、系统拓朴设计**

ES集群在架构拓朴时，一般都会采用Hot-Warm的架构模式，即设置3种不同类型的节点：Master节点、Hot 节点和Warm节点。

## **Master节点设置**：

一般会设置３个专用的maste节点，以提供最好的弹性扩展能力。当然，必须注意

discovery.zen.minimum_master_nodes属性的设置，以防split-brain问题，使用公式设置：N/2+1(N为候选master节点数）。 该节点保持: node.data: false ; 因为master节点不参与查询、索引操作，仅负责对于集群管理，所以在CPU、内存、磁盘配置上，都可以比数据节点低很多。

## **Hot节点设置**：

索引节点（写节点），同时保持近期频繁使用的索引。 属于IO和CPU密集型操作，建议使用SSD的磁盘类型，保持良好的写性能；节点的数量设置一般是大于等于3个。将节点设置为hot类型：

**node.attr.box_type:hot**

针对index, 通过设置

index.routing.allocation.require.box_type：hot 可以设置将索引写入hot节点。

## **Warm节点设置**： 

用于不经常访问的read-only索引。由于不经常访问，一般使用普通的磁盘即可。内存、CPU的配置跟Hot节点保持一致即可；节点数量一般也是大于等于3个。

当索引不再被频繁查询时，可通过

index.routing.allocation.require.box_type：warm， 将索引标记为warm, 从而保证索引不写入hot节点，以便将SSD磁盘资源用在刀刃上。一旦设置这个属性，ES会自动将索引合并到warm节点。同时，也可以在elasticsearch.yml中设置index.codec: best_compression 保证warm 节点的压缩配置。

## **Coordinating节点**：

协调节点用于做分布式里的协调，将各分片或节点返回的数据整合后返回。在ES集群中，所有的节点都有可能是协调节点，但是，可以通过设置node.master、node.data 、node.ingest 都为false 来设置专门的协调节点。需要较好的CPU和较高的内存。

# **三、ES的内存设置**

由于ES构建基于lucene, 而lucene设计强大之处在于lucene能够很好的利用操作系统内存来缓存索引数据，以提供快速的查询性能。lucene的索引文件segements是存储在单文件中的，并且不可变，对于OS来说，能够很友好地将索引文件保持在cache中，以便快速访问；因此，我们很有必要将一半的物理内存留给lucene ; 另一半的物理内存留给ES（JVM heap )。所以， 在ES内存设置方面，可以遵循以下原则：

1.当机器内存小于64G时，遵循通用的原则，50%给ES，50%留给lucene。

2.当机器内存大于64G时，遵循以下原则：

a. 如果主要的使用场景是全文检索, 那么建议给ES Heap分配4~32G的内存即可；其它内存留给操作系统, 供lucene使用（segments cache), 以提供更快的查询性能。

b. 如果主要的使用场景是聚合或排序， 并且大多数是numerics, dates, geo_points 以及not_analyzed的字符类型， 建议分配给ES Heap分配4~32G的内存即可，其它内存留给操作系统，供lucene使用(doc values cache)，提供快速的基于文档的聚类、排序性能。

c. 如果使用场景是聚合或排序，并且都是基于analyzed 字符数据，这时需要更多的heap size, 建议机器上运行多ES实例，每个实例保持不超过50%的ES heap设置(但不超过32G，堆内存设置32G以下时，JVM使用对象指标压缩技巧节省空间)，50%以上留给lucene。

3.禁止swap，一旦允许内存与磁盘的交换，会引起致命的性能问题。

通过在elasticsearch.yml中bootstrap.memory_lock: true，以保持JVM锁定内存，保证ES的性能。

4. GC设置原则：

a. 保持GC的现有设置，默认设置为：Concurrent-Mark and Sweep (CMS)，别换成G1GC，因为目前G1还有很多BUG。

b. 保持线程池的现有设置，目前ES的线程池较1.X有了较多优化设置，保持现状即可；默认线程池大小等于CPU核心数。如果一定要改，按公式（（CPU核心数* 3）/ 2）+ 1 设置；不能超过CPU核心数的2倍；但是不建议修改默认配置，否则会对CPU造成硬伤。

# **四、 集群分片设置**

ES一旦创建好索引后，就无法调整分片的设置，而在ES中，一个分片实际上对应一个lucene 索引，而lucene索引的读写会占用很多的系统资源，因此，分片数不能设置过大；所以，在创建索引时，合理配置分片数是非常重要的。一般来说，我们遵循一些原则：

1.控制每个分片占用的硬盘容量不超过ES的最大JVM的堆空间设置（一般设置不超过32G，参加上文的JVM设置原则），因此，如果索引的总容量在500G左右，那分片大小在16个左右即可；当然，最好同时考虑原则2。

2.考虑一下node数量，一般一个节点有时候就是一台物理机，如果分片数过多，大大超过了节点数，很可能会导致一个节点上存在多个分片，一旦该节点故障，即使保持了1个以上的副本，同样有可能会导致数据丢失，集群无法恢复。所以， 一般都设置分片数不超过节点数的3倍。

# **五、Mapping建模**

1.尽量避免使用nested或parent/child，能不用就不用；nested query慢，parent/child query 更慢，比nested query慢上百倍；因此能在mapping设计阶段搞定的（大宽表设计或采用比较smart的数据结构），就不要用父子关系的mapping。

2.如果一定要使用nested fields，保证nested fields字段不能过多，目前ES默认限制是50。参考：

**index.mapping.nested_fields.limit ：50**

因为针对1个document, 每一个nested field, 都会生成一个独立的document, 这将使Doc数量剧增，影响查询效率，尤其是JOIN的效率。

3.避免使用动态值作字段(key), 动态递增的mapping，会导致集群崩溃；同样，也需要控制字段的数量，业务中不使用的字段，就不要索引。控制索引的字段数量、mapping深度、索引字段的类型，对于ES的性能优化是重中之重。以下是ES关于字段数、mapping深度的一些默认设置：

**index.mapping.nested_objects.limit :10000**

**index.mapping.total_fields.limit:1000**

**index.mapping.depth.limit: 20**

# **六、 索引优化设置**

1.设置refresh_interval为-1，同时设置number_of_replicas 为0，通过关闭refresh间隔周期，同时不设置副本来提高写性能。

2.修改index_buffer_size 的设置，可以设置成百分数，也可设置成具体的大小，大小可根据集群的规模做不同的设置测试。

**indices.memory.index_buffer_size：10%（默认）**

**indices.memory.min_index_buffer_size： 48mb（默认）**

**indices.memory.max_index_buffer_size**

3.修改translog相关的设置：

a. 控制数据从内存到硬盘的操作频率，以减少硬盘IO。可将sync_interval的时间设置大一些。

**index.translog.sync_interval：5s(默认)**

b. 控制tranlog数据块的大小，达到threshold大小时，才会flush到lucene索引文件。

**index.translog.flush_threshold_size：512mb(默认)**

\4. _id字段的使用，应尽可能避免自定义_id, 以避免针对ID的版本管理；建议使用ES的默认ID生成策略或使用数字类型ID做为主键。

\5. _all字段及_source字段的使用，应该注意场景和需要，_all字段包含了所有的索引字段，方便做全文检索，如果无此需求，可以禁用；_source存储了原始的document内容，如果没有获取原始文档数据的需求，可通过设置includes、excludes 属性来定义放入_source的字段。

6.合理的配置使用index属性，analyzed 和not_analyzed，根据业务需求来控制字段是否分词或不分词。只有groupby需求的字段，配置时就设置成not_analyzed, 以提高查询或聚类的效率。

# **七、 查询优化**

\1. query_string 或multi_match的查询字段越多， 查询越慢。可以在mapping阶段，利用copy_to属性将多字段的值索引到一个新字段，multi_match时，用新的字段查询。

2.日期字段的查询， 尤其是用now 的查询实际上是不存在缓存的，因此， 可以从业务的角度来考虑是否一定要用now, 毕竟利用query cache 是能够大大提高查询效率的。

3.查询结果集的大小不能随意设置成大得离谱的值， 如query.setSize不能设置成 Integer.MAX_VALUE， 因为ES内部需要建立一个数据结构来放指定大小的结果集数据。

4.尽量避免使用script，万不得已需要使用的话，选择painless & experssions 引擎。一旦使用script查询，一定要注意控制返回，千万不要有死循环（如下错误的例子），因为ES没有脚本运行的超时控制，只要当前的脚本没执行完，该查询会一直阻塞。

![](assets\20190522220153.jpg)



5.避免层级过深的聚合查询， 层级过深的group by , 会导致内存、CPU消耗，建议在服务层通过程序来组装业务，也可以通过pipeline的方式来优化。

6.复用预索引数据方式来提高AGG性能：如通过terms aggregations 替代range aggregations， 如要根据年龄来分组，分组目标是: 少年（14岁以下） 青年（14-28） 中年（29-50） 老年（51以上）， 可以在索引的时候设置一个age_group字段，预先将数据进行分类。从而不用按age来做range aggregations, 通过age_group字段就可以了。

\7. Cache的设置及使用：

a.QueryCache: ES查询的时候，使用filter查询会使用query cache, 如果业务场景中的过滤查询比较多，建议将querycache设置大一些，以提高查询速度。

**indices.queries.cache.size： 10%（默认），可设置成百分比，也可设置成具体值，如256mb。**

当然也可以禁用查询缓存（默认是开启）， 通过index.queries.cache.enabled：false设置。

b.FieldDataCache: 在聚类或排序时，field data cache会使用频繁，因此，设置字段数据缓存的大小，在聚类或排序场景较多的情形下很有必要，可通过

indices.fielddata.cache.size：30% 或具体值10GB来设置。但是如果场景或数据变更比较频繁，设置cache并不是好的做法，因为缓存加载的开销也是特别大的。

c.ShardRequestCache: 查询请求发起后，每个分片会将结果返回给协调节点(Coordinating Node), 由协调节点将结果整合。

如果有需求，可以设置开启; 通过设置

index.requests.cache.enable: true来开启。

不过，shard request cache只缓存hits.total, aggregations, suggestions类型的数据，并不会缓存hits的内容。也可以通过设置

indices.requests.cache.size: 1%*（默认）*来控制缓存空间大小。



 

