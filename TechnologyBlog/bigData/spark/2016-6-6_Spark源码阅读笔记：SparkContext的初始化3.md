# 2016-6-6 Spark源码阅读笔记：SparkContext的初始化3
看书读源码个人笔记，以代码阅读过程中的简记为主，点到为止，加快速度，意会即可
Spark 版本：1.6.1
-----------------

### Hadoop相关配置的加载

#### 加载Hadoop相关配置信息
如果是yarn，则用YarnSparkHadoopUtil.newConfiguration。就是读取配置文件，fs.s3.|spark.hadoop.|spark.buffer.size

#### 加载外部jar文件和file文件

	_jars = _conf.getOption("spark.jars").map(_.split(",")).map(_.filter(_.size != 0)).toSeq.flatten
	_files = _conf.getOption("spark.files").map(_.split(",")).map(_.filter(_.size != 0))
	...
	if (jars != null) {
      jars.foreach(addJar)
      // addJar方法就是根据不同的情况，比如yarn等，可以是很多类型的路径，其中local是指worker中的每个本地路径
      // env.rpcEnv.fileServer.addJar(new File(uri.getPath))
      // fileServer分别有http--，netty--，AkkaFileServer三种
      // 相应地会生成k-v,key是 uri.getScheme，value是System.currentTimeMillis，然后更新环境变量到 listerBus
    }
    // addFile与addJar底层实现机制差不多啊
    if (files != null) {
      files.foreach(addFile)
      // addFile是被job中的每个节点都下载的路径
    }

	注：
	高级依赖管理
	通过spark-submit提交应用时，application jar和–jars选项中的jar包都会被自动传到集群上。Spark支持以下URL协议，并采用不同的分发策略：

	file: – 文件绝对路径，并且file:/URI是通过驱动器的HTTP文件服务器来下载的，每个执行器都从驱动器的HTTP server拉取这些文件。
	hdfs:, http:, https:, ftp: – 设置这些参数后，Spark将会从指定的URI位置下载所需的文件和jar包。
	local: –  local:/ 打头的URI用于指定在每个工作节点上都能访问到的本地或共享文件。这意味着，不会占用网络IO，特别是对一些大文件或jar包，最好使用这种方式，当然，你需要把文件推送到每个工作节点上，或者通过NFS和GlusterFS共享文件。
	注意，每个SparkContext对应的jar包和文件都需要拷贝到所对应执行器的工作目录下。一段时间之后，这些文件可能会占用相当多的磁盘。在YARN上，这些清理工作是自动完成的；而在Spark独立部署时，这种自动清理需要配置 spark.worker.cleanup.appDataTtl 属性。

#### 加载Executor环境变量executorEnvs
由master加载后发送给worker，然后worker用这个参数启动Executor

#### 启动心跳接收器_heartbeatReceiver
用于master与worker之间的相互心跳通信

#### 创建任务调度器_taskScheduler

    private var _taskScheduler: TaskScheduler = _
    ...

    // Create and start the scheduler
    // 调用createTaskScheduler方法启动
    val (sched, ts) = SparkContext.createTaskScheduler(this, master)
    _schedulerBackend = sched
    _taskScheduler = ts
    _dagScheduler = new DAGScheduler(this)
    _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)

    // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
    // constructor
    _taskScheduler.start()

    ...
    // 然后会匹配不同的模式，创建不同的scheduler和backend，然后 scheduler.initialize(backend)

    master match {
      case "local" =>
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalBackend(sc.getConf, scheduler, 1)
        scheduler.initialize(backend)
        (backend, scheduler)

#### 创建和启动DAGScheduler

创建Job，将DAG中的RDD划分到不同Stage，然后提交Stage给TaskSchedulerImpl，让它提交任务

    @volatile private var _dagScheduler: DAGScheduler = _
    。。。
    _dagScheduler = new DAGScheduler(this)

DAGScheduler主要维护了jobID与stageID之间的关系，以及shuffle的一些数据结构：

	private[spark] val metricsSource: DAGSchedulerSource = new DAGSchedulerSource(this)

	private[scheduler] val nextJobId = new AtomicInteger(0)
	private[scheduler] def numTotalJobs: Int = nextJobId.get()
	private val nextStageId = new AtomicInteger(0)

	private[scheduler] val jobIdToStageIds = new HashMap[Int, HashSet[Int]]
	private[scheduler] val stageIdToStage = new HashMap[Int, Stage]
	private[scheduler] val shuffleToMapStage = new HashMap[Int, ShuffleMapStage]
	private[scheduler] val jobIdToActiveJob = new HashMap[Int, ActiveJob]

	// Stages we need to run whose parents aren't done
	private[scheduler] val waitingStages = new HashSet[Stage]

	// Stages we are running right now
	private[scheduler] val runningStages = new HashSet[Stage]

	// Stages that must be resubmitted due to fetch failures
	private[scheduler] val failedStages = new HashSet[Stage]

	private[scheduler] val activeJobs = new mutable.HashSet[ActiveJob]
	。。。

	主要调用 DAGSchedulerEventProcessLoop 这个内部类，它继承自EventLoop
	/**
	* An event loop to receive events from the caller and process all events in the event thread. It
	* will start an exclusive event thread to process all events.
	*
	* Note: The event queue will grow indefinitely. So subclasses should make sure `onReceive` can
	* handle events in time to avoid the potential OOM.
	*/

	// 有一个 eventQueue 和一个 eventThread，这个eventThread作为后台线程循环从eventThread中取出even，调用onReceive(event)方法
	private val eventQueue: BlockingQueue[E] = new LinkedBlockingDeque[E]()

	private val stopped = new AtomicBoolean(false)

	private val eventThread = new Thread(name) {
		setDaemon(true)

	    override def run(): Unit = {
	      try {
	        while (!stopped.get) {
	          val event = eventQueue.take()
	          try {
	            onReceive(event)
	          } catch {
	            case NonFatal(e) => {
	              try {
	                onError(e)
	              } catch {
	                case NonFatal(e) => logError("Unexpected error in " + name, e)
	              }
	            }
	          }
	        }
	      } catch {
	        case ie: InterruptedException => // exit even if eventQueue is not empty
	        case NonFatal(e) => logError("Unexpected error in " + name, e)
	      }
	    }
	}


	DAGSchedulerEventProcessLoop重写了onReceive(event)方法，调用doOnReceive(event)

	/**
	   * The main event loop of the DAG scheduler.
	   */
	  override def onReceive(event: DAGSchedulerEvent): Unit = {
	    val timerContext = timer.time()
	    try {
	      doOnReceive(event)
	    } finally {
	      timerContext.stop()
	    }
	  }

	然后doOnReceive(event)处理各种事件：

	private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
    case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)

    case MapStageSubmitted(jobId, dependency, callSite, listener, properties) =>
      dagScheduler.handleMapStageSubmitted(jobId, dependency, callSite, listener, properties)

    case StageCancelled(stageId) =>
      dagScheduler.handleStageCancellation(stageId)

      。。。

#### TaskScheduler的启动

	_taskScheduler.start()
	创建并启动localEndpoint，然后放入SparkListenerExecutorAdded事件到listenerBus，
	其localEndpoint中创建并注册了Executor

#### 测量系统MetricsSystem的启动

首先在_env（SparkEnv）中定义了相应的 MetricsSystem:如果是driver端只定义不start，如果是Executor端的话会启动

    // SparkEnv.scala
	val metricsSystem = if (isDriver) {
      // Don't start metrics system right now for Driver.
      // We need to wait for the task scheduler to give us an app ID.
      // Then we can start the metrics system.
      MetricsSystem.createMetricsSystem("driver", conf, securityManager)
    } else {
      // We need to set the executor ID before the MetricsSystem is created because sources and
      // sinks specified in the metrics configuration file will want to incorporate this executor's
      // ID into the metrics they report.
      conf.set("spark.executor.id", executorId)
      val ms = MetricsSystem.createMetricsSystem("executor", conf, securityManager)
      ms.start()
      ms
    }

然后在SparkContext中启动，步骤为：
1. 绑定UI(Jetty)，2. 注册Source，3. 注册Sink，
代码如下：

    // SparkContext.scala

    def metricsSystem: MetricsSystem = if (_env != null) _env.metricsSystem else null
    ...
    // The metrics system for Driver need to be set spark.app.id to app ID.
    // So it should start after we get app ID from the task scheduler and set spark.app.id.
    metricsSystem.start()
    // Attach the driver metrics servlet handler to the web ui after the metrics system is started.
    metricsSystem.getServletHandlers.foreach(handler => ui.foreach(_.attachHandler(handler)))
    。。。
    _env.metricsSystem.registerSource(_dagScheduler.metricsSource)
    _env.metricsSystem.registerSource(new BlockManagerSource(_env.blockManager))
    _executorAllocationManager.foreach { e =>
      _env.metricsSystem.registerSource(e.executorAllocationManagerSource)
    }

#### 创建EventLoggingListener
废话少说上代码：

	/**
	 * A SparkListener that logs events to persistent storage.
	 *
	 * Event logging is specified by the following configurable parameters:
	 *   spark.eventLog.enabled - Whether event logging is enabled.
	 *   spark.eventLog.compress - Whether to compress logged events
	 *   spark.eventLog.overwrite - Whether to overwrite any existing files.
	 *   spark.eventLog.dir - Path to the directory in which events are logged.
	 *   spark.eventLog.buffer.kb - Buffer size to use when writing to output streams
	 */

	 _eventLogger =
      if (isEventLogEnabled) {
        val logger =
          new EventLoggingListener(_applicationId, _applicationAttemptId, _eventLogDir.get,
            _conf, _hadoopConfiguration)
        logger.start()
        listenerBus.addListener(logger)
        Some(logger)
      } else {
        None
      }

#### 创建和启动ExecutorAllocationManager
只有在dynamicAllocationEnabled为true的情况下开启，默认不开启使用，用于对已分配的Executor进行管理；
作用：设置动态分配最小的Executor数量，最大的Executor数量，每个Executor可以运行的Task数量等配置信息，并对配置信息进行校验


#### 创建与启动ContextCleaner
ContextCleaner工作原理和listenerBus一样，也采用监听器模式。用于清理那些超出应用范围的RDD， ShuffleDependency 和 Broadcast 以及Checkpoint等

#### Spark环境更新与ListenerBus的启动

	setupAndStartListenerBus()
    postEnvironmentUpdate()
    postApplicationStart()

最后：
#### 将SparkContext标记为激活

	  // In order to prevent multiple SparkContexts from being active at the same time, mark this
	  // context as having finished construction.
	  // NOTE: this must be placed at the end of the SparkContext constructor.
	  SparkContext.setActiveContext(this, allowMultipleContexts)