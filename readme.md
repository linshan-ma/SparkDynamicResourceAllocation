Spark动态资源调度方案
背景：
发现yarn_streamsql集群运行的都是一些sql，有时出现oom资源不够、并行度不够的情形，而用户又不知道具体的设置方法，所以用此方法，让spark动态资源调整，解决此问题。

步骤：
1、将spark目录下的spark-<version>-yarn-shuffle.jar（$SPARK_HOME/yarn/spark-2.1.0-yarn-shuffle.jar）copy到hadoop集群的所有节点中，如：$HADOOP_HOME/yarn/lib/
（可以copy到在集群的每个NodeManager上，但是建议copy到所有节点）

2、修改YARN NodeManger配置yarn-site.xml，并copy至各个节点
<property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle,spark_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services.spark_shuffle.class</name>
    <value>org.apache.spark.network.yarn.YarnShuffleService</value>
  </property>
<property>
    <name>spark.shuffle.service.port</name>
    <value>7337</value>
</property>

备注：

可以另外加上这个参数：spark.yarn.shuffle.stopOnFailure
spark.yarn.shuffle.stopOnFailure ，默认是false。当SparkShuffleService初始化失败的时候是否关闭NodeManager。这个配置是避免当机器上的SparkShuffleService不可用的时候，任务被分配到了这台机器上。

3、通过在$HADOOP_HOME/etc/hadoop /yarn-env.sh中设置YARN_HEAPSIZE（默认为1000）来增加NodeManager的堆大小，可以设置大一点，这个设置避免Shuffle过程中的内存回收
4、重启集群所有NodeManager
5、配置spark-defaults.conf
配置如下：
spark.shuffle.service.enabled true   //启用External shuffle Service服务
spark.shuffle.service.port  7337   //Shuffle Service服务端口，必须和yarn-site中的一致
spark.dynamicAllocation.enabled true  //开启动态资源分配
spark.dynamicAllocation.minExecutors 1  //每个Application最小分配的executor数
spark.dynamicAllocation.maxExecutors 30  //每个Application最大并发分配的executor数
spark.dynamicAllocation.schedulerBacklogTimeout 1s //这个参数的默认值是1秒，即当任务调度延迟超过1秒的时候，会请求增加executor
spark.dynamicAllocation.sustainedSchedulerBacklogTimeout 5s //当schedulerBacklogTimeout参数检测过一次，增加了executor，第二次检测任务调度延迟超过5秒的时候（5秒为此参数设置），再次增加executor