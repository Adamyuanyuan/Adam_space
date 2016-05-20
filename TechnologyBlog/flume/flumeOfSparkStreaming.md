# flume对Spark Streaming数据的汇集
-----------------------

### ———— wangxiaogang 2016.03

### 背景

Spark Streaming产生的数据如果直接写入hdfs中的话，对于每个读入的小文件，并行操作这些hdfs的每个slaver都会写一个小文件，这将产生大量的小文件的问题。为了解决这个问题，经过大量的调研，出于可行性与复杂性的考虑。最终决定暂时使用flume对Streaming的输出数据做一个统一的汇集，统一输出到hdfs中。

Flume NG是一个分布式、可靠、可用的系统，它能够将不同数据源的海量日志数据进行高效收集、聚合、移动，最后存储到一个中心化数据存储系统中，容易适应各种方式日志收集，并支持failover和负载均衡。

（值得一提的是，业界主流做法是用flume从日志数据的源头进行收集，然后将数据发送给Streaming或hdfs中，所以在此基础上我提出了一个新的基于flume的日志收集框架，与国信的设计从一开始就不同，可能对于简单的文件名稽核等操作，甚至不会使用到Streaming或者strom等流式处理工具来分发日志。在此只是提出）

目前还是通过flume对Streaming的数据进行汇集，因为没有查到有其它公司做过类似的事情，因此我将汇集开发的过程记录在此，以备以后查阅。

### flume学习

	bin/flume-ng agent --conf conf --conf-file example.conf --name a1 -Dflume.root.logger=INFO,console
	telnet localhost 8444

最新版本的flume有扩充zk的功能，不过目前只处于试用阶段

一些相关学习链接：

[flume 官网文档](https://flume.apache.org/FlumeDeveloperGuide.html)

[网友高手：随笔分类 - Flume-NG](http://www.cnblogs.com/lxf20061900/category/565688.html)

[flume学习系列文章](http://www.aboutyun.com/thread-12039-1-1.html)

[flume案例大全](http://www.jb51.net/article/53542.htm)

[flume分层架构配置实践](http://shiyanjun.cn/archives/category/opensource/flume)

[Flume Hdfs Sink](http://bit1129.iteye.com/category/333768)

[aboutyun关于flume专栏](http://www.aboutyun.com/forum-145-1.html)

[flume Hdfs参数配置说明](http://lxw1234.com/archives/2015/10/527.htm)


### Demo开发笔记

#### flume配置(持续更新)

假设flume已经安装配置好了，然后使用如下配置，从avro源读取数据，通过memory channel，sink到HDFS中

    conf/avroToHdfs.conf

	# Define a source, channel, sink
	a1.sources = r1
	a1.channels = c1
	a1.sinks = k1

	# Configure channel
	a1.channels.c1.type = memory
	# a1.channels.c1.capacity = 1000000
	# a1.channels.c1.transactionCapacity = 500000

	# Define an Avro source called r1 on a1 and tell it
	# to bind to 0.0.0.0:8141. Connect it to channel c1.
	a1.sources.r1.channels = c1
	a1.sources.r1.type = avro
	a1.sources.r1.bind = 0.0.0.0
	a1.sources.r1.port = 8141

	# Define a logger sink that simply logs all events it receives
	# and connect it to the other end of the same channel.
	a1.sinks.k1.channel = c1
	a1.sinks.k1.type = hdfs
	a1.sinks.k1.hdfs.path = hdfs://10.142.78.50:8020/data/testFlume/%{filePath}
	a1.sinks.k1.hdfs.filePrefix = %{fileName}
	a1.sinks.k1.hdfs.fileSuffix = .gz
	# 134217728 = 1024*1024*128

	# a1.sinks.k1.hdfs.rollSize = 1048576
	# a1.sinks.k1.hdfs.rollInterval = 0
	a1.sinks.k1.hdfs.idleTimeout = 120
	# a1.sinks.k1.hdfs.rollCount = 0
	# a1.sinks.k1.hdfs.batchSize = 1500
	a1.sinks.k1.hdfs.round = true
	a1.sinks.k1.hdfs.roundValue = 1
	a1.sinks.k1.hdfs.roundUnit = minute

	# a1.sinks.k1.hdfs.threadsPoolSize = 25
	a1.sinks.k1.hdfs.useLocalTimeStamp = true

	# a1.sinks.k1.hdfs.minBlockReplicas = 1

	a1.sinks.k1.hdfs.codeC = gzip
	a1.sinks.k1.hdfs.fileType = CompressedStream
	# a1.sinks.k1.hdfs.writeFormat = TEXT


然后执行下面代码启动flume：

	bin/flume-ng agent --conf conf --conf-file conf/avroToHdfs2.conf --name a1 -Dflume.root.logger=INFO,console

或者：

	nohup bin/flume-ng agent --conf conf --conf-file conf/avroToHdfs2.conf --name a1 -Dflume.monitoring.type=http -Dflume.monitoring.port=8533 1>/dev/null 2>&1 &

    bin/flume-ng agent --conf conf --conf-file conf/avroToHdfs2.conf --name a1 -Dflume.monitoring.type=http -Dflume.monitoring.port=8533 -Dflume.root.logger=INFO,console

#### Streaming部分flume发送客户端

    // FlumeClient.scala
	package dao

	import java.nio.charset.Charset
	import java.util
	import java.util.Map

	import com.typesafe.config.ConfigFactory
	import org.apache.flume.{EventDeliveryException, Event}
	import org.apache.flume.api.{RpcClientFactory, RpcClient}
	import org.apache.flume.event.EventBuilder

	/**
	  * Created by Adam on 2016/3/31.
	  * 将Streaming中数据通过Avro格式发送给Flume的实现类
	  */
	class FlumeClient {
	  private var client: RpcClient = null
	  private var hostname: String = null
	  private var port: Int = 0

	  def init(hostname: String, port: Int) {
	    // Setup the RPC connection
	    this.hostname = hostname
	    this.port = port
	    this.client = RpcClientFactory.getDefaultInstance(hostname, port)
	  }

	  /**
	    * 发送avro协议的数据到指定的flume中
	    *
	    * @param header 保存的是要保存的hdfs路径信息
	    * @param data   发送的数据
	    */
	  def sendDataToFlume(header: Map[String, String], data: String): Unit = {
	    //    val fileNameHeader: Map[String, String] = new HashMap[_, _]
	    //    fileNameHeader.put("fileName", "03_100_1_00_20160306115317_00011")
	    //    fileNameHeader.put("filePath", "/data/hjpt/edm/ilpf/err_new/itf_3gdpi_mbl/shanghai/20160214/00/")

	    // Create a Flume Event object that encapsulates the sample data
	    val event: Event = EventBuilder.withBody(data, Charset.forName("UTF-8"), header)
	    try {
	      client.append(event)
	    } catch {
	      case e: EventDeliveryException => {
	        client.close()
	        client = null
	        client = RpcClientFactory.getDefaultInstance(hostname, port)
	      }
	    }
	  }

	  def cleanUp(): Unit = {
	    // Close the RPC connection
	    client.close()
	  }
	}

	object ClientTest {
	  def main(args: Array[String]) : Unit = {
	    val clientTest = new FlumeClient

	    // Initialize client with the remote Flume agent's host and port
	    val hostname =  ConfigFactory.load().getString("flume.hostname")
	    val port =  ConfigFactory.load().getInt("flume.port")

	    clientTest.init(hostname, port)

	    // Send 10 events to the remote Flume agent. That agent should be
	    // configured to listen with an AvroSource.

	    val header = new util.HashMap[String, String]()
	    header.put("fileName", "03_100_1_00_20160306115317_0002")
	    header.put("filePath", "/data/hjpt/edm/ilpf/err_new/itf_3gdpi_mbl/shanghai/20160214/00/")
	    val sampleData: String = "Hello avro Flume!"
	    for (i <- 1 to 10 ) {
	      clientTest.sendDataToFlume(header, sampleData)
	    }
	    clientTest.cleanUp()

	  }

	}

运行上述发送测试程序后，就会发现在hdfs中创建了文件

	/data/testFlume/data/hjpt/edm/ilpf/err_new/itf_3gdpi_mbl/shanghai/20160214/00/03_100_1_00_20160306115317_0002.1459476611241.gz


#### 需要解决的问题

目前基本的从avro到hdfs的路径已经打通，但还有如下几个问题需要解决：

##### 问题一：文件名的问题(目前已经基本解决)

1. 因为从Streaming发送过来的数据文件路径与文件名是有比较复杂的规定的，需要通过省份，日期，时间，dpi类型，是否正确等情况定义，如果从flume端通过正则匹配的方法进行文件名命名是比较不可行的，因此需要有文件名传输机制。

路径格式：

    正确输出路径：
	/data/hjpt/edm/ilpf/comm_new/itf_3gdpi_mbl/shanghai/20160214/00/03_100_1_00_20160306115317_00000.gz

	# hdfs_path + "/data/hjpt/edm/ilpf/comm_new/"+table_id+"/"+prov_dsc+"/"+file_day+"/"+file_hour+"/"+file_name+rndnum
	错误输出路径：
	/data/hjpt/edm/ilpf/err_new/itf_3gdpi_mbl/shanghai/20160214/00/03_100_1_00_20160306115317_00000.gz

解决方案1. 通过配置多个flume agent，然后在Streaming端通过模式匹配来分发到不同的flume agent。
如果使用这种方式的话，首先将需要一百个左右的source源，然后文件名也是未知的，所以说，这个方案不可取；

解决方案2. avro的header传送文件名，然后解析header中的内容，得到文件路径。
这个方案比较符合项目要求，目前demo已经实现。

##### 问题二：关于一些性能问题需要注意的(持续调优)

http://www.w2bc.com/Article/8017

- 1. 内置hdfs sink的解析时间戳来设置目录或者文件前缀非常损耗性能，因为是基于正则来匹配的，可以通过修改源码来替换解析时间功能来极大提升性能
- 2. avro sink的batch-size可以设置大一点，默认是100，增大会减少RPC次数，提高性能； 
- 3. 关于监控monitor,后期需要监控部分的逻辑
- 4. 采集节点建议使用新的复合类型的SpillableMemoryChannel，汇总节点建议采用memory channel，具体还要看实际的数据量，一般每分钟数据量超过120MB大小的flume agent都建议用memory channel