LiveListenerBus继承自SparkListenerBus，而SparkListenerBus又继承自ListenerBus。这里有两层意思，所谓的Bus,就是一个类似总线的功能，将异步的SparkListenerEvents传递给相应的Listener,而各类Listener类似于网络通讯的监听，对各类SparkListenerEvents进行处理，只不过这些消息不是由socket而来。

ListenerBus首先维护了一个Listener类型的CopyOnWriteArrayList，并通过addListener接口供外部调用。

在SparkContext的初始化过程中，创建了LiveListenerBus的实例，其变量名是listenerBus。同事加入了一个JobProgressListener到listenerBus。在HeartbeatReceiver的初始化时，也通过SparkContext的addSparkListener，将自己加入到listenerBus。另外SparkUI也会往里面增加数个listener实例。

数个Listener实例可以处理各自关心的数据，当然，如果一条消息，不同的Listener实例都关心，那么这条消息会被多Listener实例处理。

在整个spark的运行过程中，都可以通过listenerBus的post函数，传递消息。 而LiveListenerBus本身会起一个线程，消费消息。

```
private val listenerThread = new Thread(name) {
    setDaemon(true)
    override def run(): Unit = Utils.tryOrStopSparkContext(sparkContext) {
      LiveListenerBus.withinListenerThread.withValue(true) {
        while (true) {
          eventLock.acquire()
          self.synchronized {
            processingEvent = true
          }
          try {
            val event = eventQueue.poll
            if (event == null) {
              // Get out of the while loop and shutdown the daemon thread
              if (!stopped.get) {
                throw new IllegalStateException("Polling `null` from eventQueue means" +
                  " the listener bus has been stopped. So `stopped` must be true")
              }
              return
            }
            //根据listeners数组中的Listener个数，逐个调用doPostEvent，具体可以再看SparkListenerBus的doPostEvent函数
            postToAll(event)
          } finally {
            self.synchronized {
              processingEvent = false
            }
          }
        }
      }
    }
  }
```
在整个代码中，我们可以通过"listenerBus.post"来搜索消息的传递。例如，在SparkContext的postApplicationStart函数，就像listenerBus传递了SparkListenerApplicationStart消息。而通过调度，SparkListener/EventLoggingListener/ApplicationEventListener等都会对这类消息进行处理，只要他们有"注册"到listenerBus中。

最后，我们可以通过btrace的一段脚本来看看具体的对象。

```
@OnMethod(
                clazz="+org.apache.spark.util.ListenerBus",
                method="addListener"
            )
        public static void m(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod,AnyType[] args) {
        print("entered " + probeClass+"."+probeMethod);
        printArray(args);
```
启动spark-shell后，可以打印出

```
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.ui.jobs.JobProgressListener@6dc2279c, ]
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.ui.env.EnvironmentListener@34652065, ]
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.storage.StorageStatusListener@3dbbed3e, ]
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.ui.exec.ExecutorsListener@56d5460f, ]
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.ui.storage.StorageListener@57f725b8, ]
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.ui.scope.RDDOperationGraphListener@4a0c512b, ]
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.HeartbeatReceiver@37b80ec7, ]
entered org.apache.spark.scheduler.LiveListenerBus.addListener[org.apache.spark.sql.SparkSession$Builder$$anon$1@51d9fd30, ]
```


