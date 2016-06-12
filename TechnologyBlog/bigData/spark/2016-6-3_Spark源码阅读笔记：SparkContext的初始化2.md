# 2016-6-6 Spark源码阅读笔记：SparkContext的初始化2：SparkUI的初始化
看书读源码个人笔记，以代码阅读过程中的简记为主，点到为止，加快速度，意会即可
Spark 版本：1.6.1
-----------------
在SparkConf，SparkEnv和metadaCleaner之后，SparkContext开始创建并初始化Spark UI；


### Spark UI大体架构说明
1. 采用事件监听机制，而非函数调用方式
2. DAGScaheduler 是产生各类SparkListenerEnv事件的源头，发送各类事件到listenerBus的事件队列中，listenerBus通过定时器将DAGScheduler产生的各类事件匹配到具体的SparkListener中，改变其中的统计监控数据，最终由SparkUI展示；
3. 常用的SparkListener的实现类有：environmentListener，storageStatusListener，executorsListener，storageListener，operationGraphListener等；


#### SparkContext中初始化JobProgressListener
_jobProgressListener监听器应该在SparkEnv创建之前来创建：

    _jobProgressListener = new JobProgressListener(_conf)
    listenerBus.addListener(jobProgressListener)
    ...
    _statusTracker = new SparkStatusTracker(this)

#### 设置SparkUI需要的各种Listener

	_ui =
      if (conf.getBoolean("spark.ui.enabled", true)) {
        Some(SparkUI.createLiveUI(this, _conf, listenerBus, _jobProgressListener,
          _env.securityManager, appName, startTime = startTime))
      } else {
        // For tests, do not enable the UI
        None
      }

    // Bind the UI before starting the task scheduler to communicate
    // the bound port to the cluster manager properly
    _ui.foreach(_.bind())

    // 然后在SparkUI.scala中createUI，启动其它各种listener

      private def create(
      sc: Option[SparkContext],
      conf: SparkConf,
      listenerBus: SparkListenerBus,
      securityManager: SecurityManager,
      appName: String,
      basePath: String = "",
      jobProgressListener: Option[JobProgressListener] = None,
      startTime: Long): SparkUI = {

    val _jobProgressListener: JobProgressListener = jobProgressListener.getOrElse {
      val listener = new JobProgressListener(conf)
      listenerBus.addListener(listener)
      listener
    }

    val environmentListener = new EnvironmentListener
    val storageStatusListener = new StorageStatusListener
    val executorsListener = new ExecutorsListener(storageStatusListener)
    val storageListener = new StorageListener(storageStatusListener)
    val operationGraphListener = new RDDOperationGraphListener(conf)

    listenerBus.addListener(environmentListener)
    listenerBus.addListener(storageStatusListener)
    listenerBus.addListener(executorsListener)
    listenerBus.addListener(storageListener)
    listenerBus.addListener(operationGraphListener)

    new SparkUI(sc, conf, securityManager, environmentListener, storageStatusListener,
      executorsListener, _jobProgressListener, storageListener, operationGraphListener,
      appName, basePath, startTime)
    }

#### 初始化SparkUI的前端等

	 /** Initialize all components of the server. */
	  def initialize() {
	    attachTab(new JobsTab(this))
	    attachTab(stagesTab)
	    attachTab(new StorageTab(this))
	    attachTab(new EnvironmentTab(this))
	    attachTab(new ExecutorsTab(this))
	    attachHandler(createStaticHandler(SparkUI.STATIC_RESOURCE_DIR, "/static"))
	    attachHandler(createRedirectHandler("/", "/jobs/", basePath = basePath))
	    attachHandler(ApiRootResource.getServletHandler(this))
	    // This should be POST only, but, the YARN AM proxy won't proxy POSTs
	    attachHandler(createRedirectHandler(
	      "/stages/stage/kill", "/stages/", stagesTab.handleKillRequest,
	      httpMethods = Set("GET", "POST")))
	  }
	  initialize()

#### SparkUI的前端展示的实现
如上，初始化的时候会`attachTab(new JobsTab(this))`和`attachHandler()`方法启动很多Tab和Handler来调用页面布局和展示`JobsTab`类会展示Job相关的状态，进度信息，包括`AllJobsPage`和`JobPage`两个界面：

	/** Web UI showing progress status of all jobs in the given SparkContext. */
	private[ui] class JobsTab(parent: SparkUI) extends SparkUITab(parent, "jobs") {
	  val sc = parent.sc
	  val killEnabled = parent.killEnabled
	  val jobProgresslistener = parent.jobProgressListener
	  val executorListener = parent.executorsListener
	  val operationGraphListener = parent.operationGraphListener

	  def isFairScheduler: Boolean =
	    jobProgresslistener.schedulingMode.exists(_ == SchedulingMode.FAIR)

	  attachPage(new AllJobsPage(this))
	  attachPage(new JobPage(this))
	}

AllJobsPage由render方法渲染，利用Listener中的统计监控数据生成Job的摘要信息，并调用 `jobsTable` 方法生成表格等html元素，最终使用如下代码封装CSS，JS等:

	UIUtils.headerSparkPage("Spark Jobs", content, parent, helpText = Some(helpText))

最终的展现原理是，调用了`JettyUtils`的createServletHandler方法，此法是先创建了一个HttpServer，然后调用Jetty的ServletContextHandler.addServlet方法启动这个Server

	 /** Create a context handler that responds to a request with the given path prefix */
	  def createServletHandler(
	      path: String,
	      servlet: HttpServlet,
	      basePath: String): ServletContextHandler = {
	    val prefixedPath = if (basePath == "" && path == "/") {
	      path
	    } else {
	      (basePath + path).stripSuffix("/")
	    }
	    val contextHandler = new ServletContextHandler
	    val holder = new ServletHolder(servlet)
	    contextHandler.setContextPath(prefixedPath)
	    contextHandler.addServlet(holder, "/")
	    contextHandler
	  }