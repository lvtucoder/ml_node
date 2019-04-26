Spark总结
=====

* ### spark中的RDD是什么，有哪些特性?
    + RDD叫做`弹性分布式数据集`,是spark最基本的数据抽象,它代表一个`不可变`,`可分区`,`元素可并行计算`的集合
    + 弹性表示: 
        1. `RDD中的数据可以存放在内存也可以在磁盘`
        1. `RDD的分区是可以改变的`
* ### spark中常用算子区别
    + map: 用于遍历RDD,将函数f应用与每一个元素,`返回新的RDD`
    + foreach: 遍历RDD,将函数f应用与每一个元素,`无返回值`     
    + mapPartition: 用于遍历操作`RDD的每一个分区`,`返回生成一个新的RDD`
    + foreachPartition: 用于遍历操作RDD中的每一个分区,`无返回结果`
    
    总结: 一般使用mapPartition或者foreachPartition算子比map和foreach更加高效,推荐使用

* ### 谈谈spark中的宽窄依赖

    RDD和它依赖的RDD(s)的关系有两种不同的类型,即窄依赖和宽依赖
    + l宽依赖: 指的是`一个父RDD`有`多个子RDD`      
    + 窄依赖: 指的是`一个父RDD`只有`一个子RDD`

* ### spark中的job, stage, task

    + job: 所谓一个job就是由一个action触发的动作,可以简单的理解为需要`执行一个rdd的action的时候就会生成一个job`
    + stage: `job的组成单位`,就是说一个job会被切分成1个或者多个stage,各个stage按照顺序依次执行 
    + task: 即stage下的一个任务执行单元,一般来说,`一个rdd有多少个partition就会有多少个task`,`因为每一个task只是处理一个partition上的数据`;`一个job的task数量是(stage的数量*task数量)的总和`
    
* ### `spark中如何划分stage`
    
    从代码的逻辑层面来说, stage划分的依据就是`宽依赖`; 根据生成的DAG有向无环图进行划分,从当前job的最后一个算子往前推,遇到宽依赖就把当前这个批次中的所有算子操作都划分成一个stage,然后继续按照这种方式再继续往前推,如遇到宽依赖又划分成一个stage,一直到最前面的一个算子.最后整个job会被划分成多个stage,而stage之间又存在依赖关系,后面的stage依赖前面的stage,也就是说只有前面的stage计算完毕后后面的stage才会运行.
    
* ### spark-submit的时候如何引入外部jar包

    1. spark-submit –jars 
    
           spark-submit --master yarn-client --jars ***.jar,***.jar(你的jar包，用逗号分隔) mysparksubmit.jar
           
    1. extraClassPath
    
        提交时在spark-default中设定参数，将所有需要的jar包考到一个文件里，然后在参数中指定该目录就可以了       
        
           spark.executor.extraClassPath=/home/hadoop/wzq_workspace/lib/*
           spark.driver.extraClassPath=/home/hadoop/wzq_workspace/lib/*
    
    1. fat-jar
    
        打包时候将所有的依赖与源代码打包到一起

* ### `spark如何防止内存溢出`
    
    + driver端的内存溢出
    
        `可以增大driver的内存参数: spark.driver.memory(default 1g)`
    + map过程产生大量的对象导致内存溢出
    
        这种内存溢出的原因是由于mao过程产生了大量的对象导致的.`针对这种问题,在不增加内存的情况下可以通过减少每个task的大小(增加partition个数)`,以便达到每个task即使产生大量的对象Executor的内存也能容的下.
        具体的做法`可以在产生大量对象的map操作之前调用repartition方法,分区成更小的块传入map`
    + 数据不平衡导致的内存溢出
    
        数据不平衡除了有可能导致内存溢出外还有可能导致性能问题,`解决办法就是调用repartition方法重新分区`
    
    + `shuffle后内存溢出`
        
        shuffle内存溢出的情况可以说都是shuffle后单个文件过大导致的,在spark中join,reduceByKey这一类型的过程都会有shuffle的过程,在shuffle的使用需要传入一个partitioner,大部分spark的shuffle操作默认的partitioner都是hashPartitioner,默认的是父RDD中最大的分区数,这个参数可以通过spark.default.parallenlism控制,该参数只对HashPartitioner有效.所以如果是别的partitioner导致的shuffle内存溢出就需要从partitioner的代码增加partitions的数量
        
    + standalone模式下资源分配不均匀导致的内存溢出
        
         在standalone的模式下如果配置了–total-executor-cores 和 –executor-memory 这两个参数，但是没有配置–executor-cores这个参数的话，就有可能导致，每个Executor的memory是一样的，但是cores的数量不同，那么在cores数量多的Executor中，由于能够同时执行多个Task，就容易导致内存溢出的情况。这种情况的解决方法就是同时配置–executor-cores或者spark.executor.cores参数，确保Executor资源分配均匀。使用rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)代替rdd.cache()。rdd.cache()和rdd.persist(Storage.MEMORY_ONLY)是等价的，在内存不足的时候rdd.cache()的数据会丢失，再次使用的时候会重算，而rdd.persist(StorageLevel.MEMORY_AND_DISK_SER)在内存不足的时候会存储在磁盘，避免重算，只是消耗点IO时间。
    
* ### spark中的cache和persist的区别
    
    + cache: 缓存数据,默认是缓存在内存中,本质也是调用了persist
    + persist: 缓存数据,有丰富的数据缓存策略(memory_only,memory_and_disk等等),.可以保存在内存中也可以保存在磁盘,使用的时候只需要指定对应的缓存级别就可以
    
* ### `spark中的数据倾斜的现象,原因,后果`

    + 数据倾斜现象
    
        多数的task执行速度较快,少数的task执行时间非常长,或者很长时间后提示内存不足,执行失败
        
    + 数据倾斜原因
                          
        - 数据问题
            1. key本身分布不平衡
            1. key的设置不合理
            
        - spark使用问题
            1. shuffle时的并发度不够
            1. 计算方式有误
    
    + 数据倾斜的后果
        
        1. spark中stage的执行时间受限于最后那个执行完成的task,因此运行缓慢的任务会拖垮整个程序的运行速度(分布式程序运行的速度室友堆满的那个task决定的)        
        1. 过多的数据在同一个task中进行,将会把executor撑爆
        
* ### `如何解决spark中的数据倾斜问题`        

    发现数据倾斜问题不要急于提高executor的资源,修改参数或者修改程序,首先要检查数据本身,是够存在异常
    + 数据本身问题造成的数据倾斜  
    
        *`找出异常的key`*  
        
        如果如果任务长时间卡在最后1个或者几个任务,首先要对key进行抽样分析,判断是哪些key造成的  
        选取key,对数据进行抽样,统计出现的次数,根据出现的次数大小排序取出前几个;如果发现多数数据分布都比较平均而个别数据比其他数据大上若干个数量级则说明发生了数据倾斜.
        
        经过分析,倾斜的数据主要有以下三种情况:  
        1. `null(空值)或是一些无意义的信息之类的,大多是这个原因引起的` 
        1. `无效数据,大量重复的测试数据或者是对结果影响不大的有效数据`
        1. `有效数据,业务导致的正常数据分布`  
        
        **解决办法:**
        
        前两种情况导致的数据倾斜直接对数据进行过滤即可(因为该数据对当前业务不会产生影响)  
        最后一种情况导致的数据倾斜则需要进行一个特殊的操作,常见的方法有以下几种:  
        1. `隔离执行, 将异常的key过滤出来单独处理,最后与正常数据的处理结果进行union操作`  
        2. `对key先添加随机数, 进行操作后,去掉随机数,再进行一个操作`
        3. `使用reduceByKey代替groupByKey(reduceByKey用于对每个key对应的多个value进行marge操作,最重要的是它能够在本地先进行merge操作,并且merge操作可以通过函数自定义)`
        4. `使用mapjoin`
        
        ***案例:***  
        如果使用reduceByKey因为数据倾斜造成运行失败的情况,具体操作流程如下:
        1. 将原始的key转化成key+随机数(例如Random.nextInt)
        2. 对数据进行reduceByKey
        3. 将key+随机数 转成key
        4. 再对数据进行reduceByKey
    
    + spark使用不当造成的数据倾斜
    
        - **`提高shuffle并行度`**    
            并行度：是指指令并行执行的最大条数。在指令流水中，同时执行多条指令称为指令并行。  
            理论上：每一个stage下有多少的分区，就有多少的task，task的数量就是我们任务的最大的并行度  
            一般情况下，`一个task运行的时候使用一个cores`
        - 使用map join 代替reduce join  
            在小表不是特别大的情况下可以使用程序避免shuffle的过程,自然也就没有了数据倾斜的问题
* ### flume整合sparkStreaming问题

    + 如何实现sparkStreaming读取flume中的数据  
        主要有两种方式:
         1. 推模式: Flume将数据Push推给SparkStreaming
         
            推模式的理解就是Flume作为缓存,存有数据.监听对应端口,如果服务可以链接就将数据push过去  
            优点: `简单,耦合低`  
            缺点: `如果SparkStreaming程序没有启动的话flume会报错,同时可能会导致SparkStreaming程序来不及消费的情况`  
                
         2. 拉模式: SparkStreaming从Flume中Poll拉去数据
            拉模式就是自己定义一个sink, SparkStreaming自己去channel里面取数据  
            优点: SparkStreaming根据自身条件去获取数据,稳定性好
         
    + 实际开发中如何保证数据不丢失的
        
        flume那边采用的channel是将数据落地到磁盘,保证数据源端安全性  
        SparkStreaming通过拉模式整合的时候,使用了FlumeUtils这样一个类,要选个保证数据不丢失,数据的准确性,可以在构建StreamingContext的时候利用        
        StreamingContext.getOrCreate（checkpoint, creatingFunc: () => StreamingContext）来创建一个StreamingContext,使用StreamingContext.getOrCreate来创建StreamingContext对象，传入的第一个参数是checkpoint的存放目录，第二参数是生成StreamingContext对象的用户自定义函数。如果checkpoint的存放目录存在，则从这个目录中生成StreamingContext对象；如果不存在，才会调用第二个函数来生成新的StreamingContext对象。在creatingFunc函数中，除了生成一个新的StreamingContext操作，还需要完成各种操作，然后调用ssc.checkpoint(checkpointDirectory)来初始化checkpoint功能，最后再返回StreamingContext对象。
        这样，在StreamingContext.getOrCreate之后，就可以直接调用start()函数来启动（或者是从中断点继续运行）流式应用了。如果有其他在启动或继续运行都要做的工作，可以在start()调用前执行
        
        流式计算中使用checkpoint的作用:  
        保存元数据,包括流式应用的配置、流式没崩溃之前定义的各种操作、未完成所有操作的batch。元数据被存储到容忍失败的存储系统上，如HDFS。这种ckeckpoint主要针对driver失败后的修复。
        保存流式数据，也是存储到容忍失败的存储系统上，如HDFS。这种ckeckpoint主要针对window operation、有状态的操作。无论是driver失败了，还是worker失败了，这种checkpoint都够快速恢复，而不需要将很长的历史数据都重新计算一遍（以便得到当前的状态）。

        设置流式数据checkpoint的周期:
        对于一个需要做checkpoint的DStream结构，可以通过调用DStream.checkpoint(checkpointInterval)来设置ckeckpoint的周期，经验上一般将这个checkpoint周期设置成batch周期的5至10倍。
        
        使用write ahead logs功能:
        这是一个可选功能，建议加上。这个功能将使得输入数据写入之前配置的checkpoint目录。这样有状态的数据可以从上一个checkpoint开始计算。开启的方法是把spark.streaming.receiver.writeAheadLogs.enable这个property设置为true。另外，由于输入RDD的默认StorageLevel是MEMORY_AND_DISK_2，即数据会在两台worker上做replication。实际上，Spark Streaming模式下，任何从网络输入数据的Receiver（如kafka、flume、socket）都会在两台机器上做数据备份。如果开启了write ahead logs的功能，建议把StorageLevel改成MEMORY_AND_DISK_SER。修改的方法是，在创建RDD时由参数传入。
        使用以上的checkpoint机制，确实可以保证数据0丢失。但是一个前提条件是，数据发送端必须要有缓存功能，这样才能保证在spark应用重启期间，数据发送端不会因为spark streaming服务不可用而把数据丢弃。而flume具备这种特性，同样kafka也具备。

    + SparkStreaming的数可靠性
    
       有了checkpoint机制 write ahead log机制, receiver缓存机制,可靠的receiver(即数据接收并备份成功后会发送ack), 可以保证无论是worker失效还是driver失效,都是数据0丢失.
       原因是: 如果没有Receiver服务的worker失效了，RDD数据可以依赖血统来重新计算；如果Receiver所在worker失败了，由于Reciever是可靠的，并有write ahead log机制，则收到的数据可以保证不丢；如果driver失败了，可以从checkpoint中恢复数据重新构建。

* ### kafka整合sparkStreaming问题    

    + 如何实现sparkStreaming读取kafka中的数据
    
        有两种方式与sparkStreaming整合:
        1. receiver:   
            是采用了kafka的高级api, 利用receiver接收器来接受kafka topic中的数据, 从kafka中接收来的数据会存储在spark的executor中,之后sparkStreaming提交的job会处理这些数据,kafka中topic的偏移量保存kafka的topic中
        2. direct:  
            不同于Receiver的方式，Direct方式没有receiver这一层，其会周期性的获取Kafka中每个topic的每个partition中的最新offsets，之后根据设定的maxRatePerPartition来处理每个batch。（设置spark.streaming.kafka.maxRatePerPartition=10000。限制每秒钟从topic的每个partition最多消费的消息条数）
            
    + 对比两种方式的优缺点
    
        - 采用receiver方式：这种方式可以保证数据不丢失，但是无法保证数据只被处理一次
        - 采用direct方式:相比Receiver模式而言能够确保机制更加健壮. 区别于使用Receiver来被动接收数据, Direct模式会周期性地主动查询Kafka, 来获得每个topic+partition的最新的offset, 从而定义每个batch的offset的范围. 当处理数据的job启动时, 就会使用Kafka的简单consumer api来获取Kafka指定offset范围的数据。

***

* ### driver的功能是什么？

    1. 一个Spark作业运行时包括一个Driver进程,也就是作业的主进程,具有main函数,并且有SparkContext的实例,是程序的入口
    2. 功能: 负责向集群申请资源,向master注册信息,负责了作业的调度,作业解析,生成stage并调度task到executor上.包含DAGScheduler，TaskScheduler
    
* ### spark有几种部署模式,每种模式有什么特点?
    
    1. 本地模式
    2. standalone模式    
    3. spark on yarn 模式
    4. mesos模式

* ### spark为什么比mapReduce快?

    1. 基于内存的计算,减少低效的磁盘交互;
    2. 高效的调度算法, 基于DAG
    3. 容错机制Linage,精华部分就是DAG和Linage
    
* ###  hadoop和spark的shuffle相同和差异?
    
    | |MapReduce|Spark|
    |----|----|----|
    |collect|在内存中构建了一块数据结构用于map输出的缓冲|没有在内存中构建缓冲而是直接输出到磁盘文件|
    |sort|map的输出有排序 |map的输出没有排序 |
    |merge |对磁盘上的多个spill文件进行合并成一个输出文件 |在map端没有merge过程,在输出时直接是对应一个reduce的数据写到一个文件中,这些文件同时存在并发写,最后不需要合并成一个 |
    |copy框架 |jetty |netty或者直接socket流 |
    |对于本节点上的文件 |仍然是通过网络框架拖取数据 |不通过网络框架,对于在本节点上的map输出文件,采用本地读取的方式 |
    |copy过来的输出存放位置 |先存放内存,内存放不下时写到磁盘 |一种方式全部存放内存;另一种方式先放内存 |
    |merge sort |最后会对磁盘文件和内存中的数据进行合并排序 |对于采用另一种方式时也会有合并排序的过程 |

    