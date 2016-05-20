## flume调优笔记

#### 调优目标
[生产环境](../master/DPIStatistics.md)一天的数据量可达20TB，高峰期每秒的数据可达每秒处理350W/s,992.89M/s，按照gz格式压缩后为 290.2M/s。目标

Flume目标：能够达到生产环境中最高的1G/s的数据落地速度，期望每个Flume Agent 能够达到17M/s的速度，这样只要60个flume agent即可满足数据落地的要求。

#### flume内存调优

默认情况下，Flume Agent进程的堆内存设置比较小，在日志数据量比较大的情况下就需要修改并调试这些参数，以满足业务需要。设置JVM相关参数，可以修改conf/flume-env.sh文件（也可以直接在启动Flume Agent服务时指定JVM选项参数），例如修改JAVA_OPTS变量，示例如下所示：

	JAVA_OPTS="-server -Xms1024m -Xmx4096m -Dcom.sun.management.jmxremote -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:ParallelGCThreads=4 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/data/flume/logs/gc-ad.log"

这样，可以方便地修改GC策略，一般由于Flume实时收集日志比较注重实时性，希望能够快速地响应，尽量减少GC导致暂停业务线程被挂起的时间，所以可以将GC设置为ParNew+CMS策略。将GC日志输出，在一定程度上能够更加方便地观察Flume Agent服务运行过程中JVM GC的详细情况，通过诊断来优化服务运行。

#### flume发送数据测量脚本

为了更加量化得统计flume agent的性能，我写了一个flume测试的脚本：`startFlumeA2HTest.py`,就是调用avro-client 客户端，然后得到计算处理时间，进而可以大概计算flume每秒处理多少兆的数据，以及每秒可以处理多少行的数据。

一个Flume中部署的 Avro 客户端，能够通过 avro RPC 机制，将一个指定的文件发送给 Flume 的 Avro Source，作为测试：

	bin/flume-ng avro-client -H localhost -p 8141 -F /usr/bdusr01/xiaogang/data/itf_3gdpi_mbl/shanghai/03_100_0_00_20160405160000_00001.gz

### 调优记录
#### 第一次调优

第一次测试：通过 flume-ng自带的avrosource，发送14M的数据
问题：

	[ERROR - org.apache.flume.source.AvroSource.appendBatch(AvroSource.java:388)] Avro source r1: Unable to process event batch. Exception follows.

原因：根据日志，是内存容量不够，因此在这里将容量设置为1G，a1.channels.c1.capacity = 1000000，发现问题确实解决了，又由于内存channel的容量默认是jvm的0.8倍，我的jvm设置为初始1G最大2G，所以0.8倍很适合我的agent，因此直接注释掉这一个选项就好了。继续测试，果然解决了这个问题；

#### 第二次

通过startFlumeA2HTest.py脚本,得到：

seconds:335 mPerSecond:3.20298507463 linesPerSecond:10161

还差很远

#### 第三次
jvm内存调优到4096M，memory.capacity = 3276800 ,也就是4G*0.8

通过startFlumeA2HTest.py脚本,得到：

seconds:335  mPerSecond:3.18397626113 linesPerSecond:10100.7121662

还差很远

#### 第四次
更改gc策略 -XX:+UseParNewGC -XX:+UseConcMarkSweepGC -XX:ParallelGCThreads=4 -verbose:gc

通过startFlumeA2HTest.py脚本,得到：

seconds:334 mPerSecond:3.2125748503 linesPerSecond:10191.4371257

快了一秒

#### 第五次
换hdfs

通过startFlumeA2HTest.py脚本,得到：

快了一秒
seconds:332 mPerSecond:3.23192771084 linesPerSecond:10252.8313253

#### 第六次
后来思考是因为数据源发送数据太慢的缘故，因此将数据源的发送速率调整到了原来的两倍

seconds:328 mPerSecond:6.54268292683 linesPerSecond:20755.7317073

快了两倍，6M/s

#### 第七次
优化gc，采用默认gc，内存调整到5G，将数据源的发送速率调整到了原来的三倍

seconds:334 mPerSecond:9.6377245509 linesPerSecond:30574.3113772

快了三倍，9M/s

#### 第八次
内存调整到6G，将数据源速度变为原来的4倍

seconds:441 mPerSecond:9.73242630385 linesPerSecond:30874.739229
性能提升并不明显，因为出现fullGC，channel拥堵，需要继续优化

#### 第九次
内存调整到8G，发送速度仍为原来的4倍
seconds:326 mPerSecond:13.1656441718 linesPerSecond:41766.1349693
速度提升到了13M/s，但channel拥堵，后期出现fullGC，需要优化GC与并发的hdfs sink优化

#### 第十次
调整多个hdfs sink，通过几次试验，先将hdfs sink数量加到 4个

seconds:264 mPerSecond:16.2575757576 linesPerSecond:51574.8484848
速度提升到了16M/s，channel并不是特别拥堵，fullGC也并没有出现，hdfs sink并发好像还有潜力的样子

#### 第十一次
调整多个hdfs sink，通过几次试验，先将hdfs sink数量加到 4个

seconds:264 mPerSecond:16.2575757576 linesPerSecond:51574.8484848
速度提升到了16M/s，channel并不是特别拥堵，fullGC也并没有出现，hdfs sink并发好像还有潜力的样子

#### 第十二次
将hdfs sink数量加到 8个，并且avro source的数量加到8个
出现 java.lang.OutOfMemoryError: unable to create new native thread 异常，也就是线程数太多了，不够用，可以解决的方法有 调小 线程栈大小Xss参数，但是最低不能低于228K，否则启动不了flume，默认是1M，

先将source的数量限制为6个，则如下：
seconds:246 mPerSecond:26.1707317073 linesPerSecond:83022.9268293

速度提升到了26M/s，channel并不是特别拥堵，fullGC也并没有出现，但是线程数因为虚拟机的限制而限制了。


#### 第十三次
为了解决上述出现的线程数的错误，将avro source的发送程序移到.55的机器上


先将source的数量限制为8个，则如下：
seconds:656 mPerSecond:13.0853658537 linesPerSecond:41511.4634146

速度下降到了13M/s，channel很空，fullGC也并没有出现，目前还不知道原因。

#### 第十四次
将source的数量增加为16个，则如下：
seconds:678 mPerSecond:25.3215339233 linesPerSecond:80328.9675516

速度回升到了25M/s，channel很空，fullGC 4分钟左右出现一次。

#### 第十五次
将source的数量增加为32个，channel很堵肯定是不行的，fullGC太频繁，8个hdfs sink已经忙不过来了，被我提前kill。

#### 第十六次
jvm内存增加到10G，任然将source的数量设置为32个，然后为了观察gc的详细情况，为gc设置了日志，channel很堵肯定是不行的，fullGC太频繁，8个hdfs sink已经忙不过来了，被我提前kill。


#### 第十七次
hdfs sink数量增加到10个，jvm内存增加到10G，仍然将source的数量设置为32个，每个source的发送速度可以达到1.5M每秒，

seconds:787 mPerSecond:43.6289707751 linesPerSecond:138406.709022

速度增加到了43.6M/s，13.8W条/s channel一点都不堵，fullGC 150s左右出现一次。

##### 截止目前。flume的速度已经达到了我预期的压缩后6M/s的目标，相当于12.75M/s

#### 第十八次
优化source的batch，使用

	sed -n 'N;N;s/\n/ /g'p 1.txt

命令将发送数据每隔3行合并成一行

seconds:792 mPerSecond:43.35 linesPerSecond:45844.28 * 3 行每秒

在我的测试中，速度在avro源没有增加的情况下并没有增加(可能是发送数据的测试程序的限制)，fullGC 150s左右出现一次。速度并没有因为原数据合并而提升。只与每秒的数据大小有关。但每个event增大后的好处是可以明显增加网络中包的个数。

#### 第十九次

将source的数量设置为48个，channel一点都不堵，fullGC 150s左右出现一次,速度为：

	seconds:973 mPerSecond:52.9331963001 linesPerSecond:55974.3144913 * 3 

	53M/s的数据处理速度, 16.8W行/s

#### 第二十次
将agent的jvm内存减少为6G，相应地更改了一些flume的参数，channel一点都不堵，fullGC 50s左右出现一次,速度为：

	seconds:1092 mPerSecond:55.0256410256 linesPerSecond:58186.974359

	55M/s的处理速度，17.4w行/s

#### 第二十一次
将agent的jvm内存减少为4G，相应地更改了一些memory channel的参数，channel一点都不堵，fullGC 30s左右出现一次,速度为：

	seconds:1110 mPerSecond:54.1333333333 linesPerSecond:57243.4018018

	54M/s的处理速度，17.2w行/s

但其中 PSPermGen 的使用量为 28.2M，预设为28.6M，使用率为98%。可以观察到之前每次程序，PSPermGen的使用量都为 30M左右，因此至少要为程序分配 30M的持久区的容量

#### 总结

现在测得在虚拟机中，我的一个flume agent具有55M/s的数据处理速度，压缩后16M/s，相当于 17W行/s的数据,已经满足要求

对于生产环境中压缩后 300M/s 的数据来说，我只需要 20个4G或6G内存的flume agent，就可以满足目前生产环境高峰期数据落地的需求。

当然，flume的速度可以继续增加，以下几个原因导致在测试环境中flume的速度比较慢：

- 1. 虚拟机的性能并不高，如果放到生产环境的物理机上，性能会更快
- 2. 数据发送测试程序的速度可以更快，现在的flume只是满足数据发送测试程序的速度，所以可能还会有一些潜力没有发挥出来

#### 继续提升的地方

- 1. JVM还有调优的空间，目前的策略是默认的GC方式，之前被我调了一下性能反而调低了
- 2. flume hdfs sink，memory channel的参数还有调优空间
- 3. 目前没有负载均衡
- 4. 最好每个agent与省份有一定的对应关系，如果n个省份同时通过一个agent，每个agent有m个hdfs sink，那同时写hdfs的文件数为 n*m,那a个agent同时打开hdfs的文件数为 a*n*m。还是要控制数量。(目前已经想到解决方案)
- 5. 后期希望增加hdfs sink数量随着数据量的增大动态地启用的优化逻辑


### 调优总结

#### 关于GC与JVM
1. 当然可以的话，内存越大越好，后来扩充到10G内存
2. 刚开始后有段时间为了追求大数据并发速度，改成了CMS的内存回收机制，发现效果反而不好，后来改成默认的内存回收机制，效果烧好一点
3. 由于之前测试flume的机器中虚拟机对线程数的限制为1024，因为测试flume的数据发送程序和flume程序在同一台机器，所以当hdfs sink数和测试发送程序的并发数增大到一定程度后，线程数被限制，为了解决这个问题，我将测试发送数据程序换到另外一台机器中
4. 我的JVM参数设置如下：
	
	export JAVA_OPTS="-Xms10g -Xmx10g -Dcom.sun.management.jmxremote -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:../gc-flume.log"


#### 关于监控monitor

flume支持http(`其实就是将统计信息jmx信息，封装成json串，使用jetty展示在浏览器中而已)，cloudera manager（前提是你得安装CDH版本）、ganglia(这个天生就是支持的)、zabbix等(美团已实现)等很多种监控，我在测试环境中使用的是最简单的http的监控。

关于http的监控，请移步我根据网上的资料和自己实践总结的[监控笔记](flumeMonitorDescription.md)