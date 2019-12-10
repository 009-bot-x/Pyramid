[LeakCanary](https://github.com/square/leakcanary/) 是由 [Square](https://github.com/square) 开源的针对 `Android` 和 `Java` 的内存泄漏检测工具。

# 使用

`LeakCanary` 的集成过程很简单，首先在 `build.gradle` 文件中添加依赖：

```
dependencies {
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:1.5.4'
  releaseImplementation 'com.squareup.leakcanary:leakcanary-android-no-op:1.5.4'
}
```

`debug` 和 `release` 版本中使用的是不同的库。`LeakCanary` 运行时会经常执行 `GC` 操作，在 `release` 版本中会影响效率。`android-no-op` 版本中基本没有逻辑实现，用于 `release` 版本。

然后实现自己的 `Application` 类：

```Java
public class ExampleApplication extends Application {

  @Override public void onCreate() {
    super.onCreate();
    if (LeakCanary.isInAnalyzerProcess(this)) {
      // This process is dedicated to LeakCanary for heap analysis.
      // You should not init your app in this process.
      return;
    }
    LeakCanary.install(this);
    // Normal app init code...
  }
}
```

这样就集成完成了。当 `LeakCanary` 检测到内存泄露时，会自动弹出 `Notification` 通知开发者发生内存泄漏的 `Activity` 和引用链，以便进行修复。

# 源码分析

从入口函数 `LeakCanary.install(this)` 开始分析：

## LeakCanary.install

`LeakCanary.java`

```Java
/**
 * Creates a {@link RefWatcher} that works out of the box, and starts watching activity
 * references (on ICS+).
 */
public static RefWatcher install(Application application) {
  return refWatcher(application).listenerServiceClass(DisplayLeakService.class)
      .excludedRefs(AndroidExcludedRefs.createAppDefaults().build())
      .buildAndInstall();
}
```

### LeakCanary.refWatcher
`LeakCanary.java`
```Java
/** Builder to create a customized {@link RefWatcher} with appropriate Android defaults. */
public static AndroidRefWatcherBuilder refWatcher(Context context) {
  return new AndroidRefWatcherBuilder(context);
}
```

`refWatcher()` 方法新建了一个 `AndroidRefWatcherBuilder` 对象，该对象继承于 `RefWatcherBuilder` 类，配置了一些默认参数，利用建造者构建一个 `RefWatcher` 对象。

### AndroidRefWatcherBuilder.listenerServiceClass
`AndroidRefWatcherBuilder.java`
```Java
public AndroidRefWatcherBuilder listenerServiceClass(
    Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
  return heapDumpListener(new ServiceHeapDumpListener(context, listenerServiceClass));
}
```

`RefWatcherBuilder.java`
```Java
/** @see HeapDump.Listener */
public final T heapDumpListener(HeapDump.Listener heapDumpListener) {
  this.heapDumpListener = heapDumpListener;
  return self();
}
```

`DisplayLeakService.java`
```Java
/**
 * Logs leak analysis results, and then shows a notification which will start {@link
 * DisplayLeakActivity}.
 *
 * You can extend this class and override {@link #afterDefaultHandling(HeapDump, AnalysisResult,
 * String)} to add custom behavior, e.g. uploading the heap dump.
 */
public class DisplayLeakService extends AbstractAnalysisResultService {}
```

`listenerServiceClass()` 方法绑定了一个后台服务 `DisplayLeakService`，这个服务主要用来分析内存泄漏结果并发送通知。你可以继承并重写这个类来进行一些自定义操作，比如上传分析结果等。

### RefWatcherBuilder.excludedRefs
`RefWatcherBuilder.java`

```Java
public final T excludedRefs(ExcludedRefs excludedRefs) {
  this.excludedRefs = excludedRefs;
  return self();
}
```

`AndroidExcludedRefs.java`

```Java
/**
 * This returns the references in the leak path that can be ignored for app developers. This
 * doesn't mean there is no memory leak, to the contrary. However, some leaks are caused by bugs
 * in AOSP or manufacturer forks of AOSP. In such cases, there is very little we can do as app
 * developers except by resorting to serious hacks, so we remove the noise caused by those leaks.
 */
public static ExcludedRefs.Builder createAppDefaults() {
  return createBuilder(EnumSet.allOf(AndroidExcludedRefs.class));
}

public static ExcludedRefs.Builder createBuilder(EnumSet<AndroidExcludedRefs> refs) {
  ExcludedRefs.Builder excluded = ExcludedRefs.builder();
  for (AndroidExcludedRefs ref : refs) {
    if (ref.applies) {
      ref.add(excluded);
      ((ExcludedRefs.BuilderWithParams) excluded).named(ref.name());
    }
  }
  return excluded;
}
```

`excludedRefs()` 方法定义了一些对于开发者可以忽略的路径，意思就是即使这里发生了内存泄漏，`LeakCanary` 也不会弹出通知。这大多是系统 Bug 导致的，无需用户进行处理。

### AndroidRefWatcherBuilder.buildAndInstall

最后调用 `buildAndInstall()` 方法构建 `RefWatcher` 实例并开始监听 `Activity` 的引用：

`AndroidRefWatcherBuilder.java`

```Java
/**
 * Creates a {@link RefWatcher} instance and starts watching activity references (on ICS+).
 */
public RefWatcher buildAndInstall() {
  RefWatcher refWatcher = build();
  if (refWatcher != DISABLED) {
    LeakCanary.enableDisplayLeakActivity(context);
    ActivityRefWatcher.install((Application) context, refWatcher);
  }
  return refWatcher;
}
```

看一下主要的 `build()` 和 `install()` 方法：

#### RefWatcherBuilder.build
`RefWatcherBuilder.java`

```Java
/** Creates a {@link RefWatcher}. */
 public final RefWatcher build() {
   if (isDisabled()) {
     return RefWatcher.DISABLED;
   }

   ExcludedRefs excludedRefs = this.excludedRefs;
   if (excludedRefs == null) {
     excludedRefs = defaultExcludedRefs();
   }

   HeapDump.Listener heapDumpListener = this.heapDumpListener;
   if (heapDumpListener == null) {
     heapDumpListener = defaultHeapDumpListener();
   }

   DebuggerControl debuggerControl = this.debuggerControl;
   if (debuggerControl == null) {
     debuggerControl = defaultDebuggerControl();
   }

   HeapDumper heapDumper = this.heapDumper;
   if (heapDumper == null) {
     heapDumper = defaultHeapDumper();
   }

   WatchExecutor watchExecutor = this.watchExecutor;
   if (watchExecutor == null) {
     watchExecutor = defaultWatchExecutor();
   }

   GcTrigger gcTrigger = this.gcTrigger;
   if (gcTrigger == null) {
     gcTrigger = defaultGcTrigger();
   }

   return new RefWatcher(watchExecutor, debuggerControl, gcTrigger, heapDumper, heapDumpListener,
           excludedRefs);
 }
```

`build()` 方法利用建造者模式构建 `RefWatcher` 实例，看一下其中的主要参数：

* `watchExecutor` : 线程控制器，在 `onDestroy()` 之后并且主线程空闲时执行内存泄漏检测
* `debuggerControl` : 判断是否处于调试模式，调试模式中不会进行内存泄漏检测
* `gcTrigger` : 用于 `GC`，`watchExecutor` 首次检测到可能的内存泄漏，会主动进行 `GC`，`GC` 之后会再检测一次，仍然泄漏的判定为内存泄漏，进行后续操作
* `heapDumper` : `dump` 内存泄漏处的 `heap` 信息，写入 `hprof` 文件
* `heapDumpListener` : 解析完 `hprof` 文件并通知 `DisplayLeakService` 弹出提醒
* `excludedRefs` : 排除可以忽略的泄漏路径

#### LeakCanary.enableDisplayLeakActivity

接下来就是最核心的 `install()` 方法，这里就开始观察 `Activity` 的引用了。在这之前还执行了一步操作，`LeakCanary.enableDisplayLeakActivity(context);` ：

```Java
public static void enableDisplayLeakActivity(Context context) {
  setEnabled(context, DisplayLeakActivity.class, true);
}
```

最后执行到 `LeakCanaryInternals#setEnabledBlocking` ：

```Java
public static void setEnabledBlocking(Context appContext, Class<?> componentClass,
    boolean enabled) {
  ComponentName component = new ComponentName(appContext, componentClass);
  PackageManager packageManager = appContext.getPackageManager();
  int newState = enabled ? COMPONENT_ENABLED_STATE_ENABLED : COMPONENT_ENABLED_STATE_DISABLED;
  // Blocks on IPC.
  packageManager.setComponentEnabledSetting(component, newState, DONT_KILL_APP);
}
```

这里启用了 `DisplayLeakActivity` 并且显示应用图标。注意，这是指的不是你自己的应用图标，是一个单独的 `LeakCanary` 的应用，用于展示内存泄露历史的，入口函数是 `DisplayLeakActivity`，在 [AndroidManifest.xml](https://github.com/square/leakcanary/blob/master/leakcanary-android/src/main/AndroidManifest.xml) 中可以看到默认情况下 `android:enabled="false"` :

```xml
<activity
    android:theme="@style/leak_canary_LeakCanary.Base"
    android:name=".internal.DisplayLeakActivity"
    android:process=":leakcanary"
    android:enabled="false"
    android:label="@string/leak_canary_display_activity_label"
    android:icon="@mipmap/leak_canary_icon"
    android:taskAffinity="com.squareup.leakcanary.${applicationId}"
    >
  <intent-filter>
    <action android:name="android.intent.action.MAIN"/>
    <category android:name="android.intent.category.LAUNCHER"/>
  </intent-filter>
</activity>
```
#### ActivityRefWatcher.install
`ActivityRefWatcher.java`

```Java
public static void install(Application application, RefWatcher refWatcher) {
  new ActivityRefWatcher(application, refWatcher).watchActivities();
}

public void watchActivities() {
  // Make sure you don't get installed twice.
  stopWatchingActivities();
  application.registerActivityLifecycleCallbacks(lifecycleCallbacks);
}
```

`watchActivities()` 方法中先解绑生命周期回调注册 `lifecycleCallbacks`，再重新绑定，避免重复绑定。`lifecycleCallbacks` 监听了 `Activity` 的各个生命周期，在 `onDestroy()` 中开始检测当前 `Activity` 的引用。

```java
private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
    new Application.ActivityLifecycleCallbacks() {
      @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
      }

      @Override public void onActivityStarted(Activity activity) {
      }

      @Override public void onActivityResumed(Activity activity) {
      }

      @Override public void onActivityPaused(Activity activity) {
      }

      @Override public void onActivityStopped(Activity activity) {
      }

      @Override public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
      }

      @Override public void onActivityDestroyed(Activity activity) {
        ActivityRefWatcher.this.onActivityDestroyed(activity);
      }
    };

    void onActivityDestroyed(Activity activity) {
      refWatcher.watch(activity);
    }
```

下面着重分析 `RefWatcher` 是如何检测 `Activity` 的。

## RefWatcher.watch

调用 `RefWatcher#watch` 检测 `Activity`。
`RefWatcher.java`
```java
/**
 * Identical to {@link #watch(Object, String)} with an empty string reference name.
 *
 * @see #watch(Object, String)
 */
public void watch(Object watchedReference) {
  watch(watchedReference, "");
}

/**
 * Watches the provided references and checks if it can be GCed. This method is non blocking,
 * the check is done on the {@link WatchExecutor} this {@link RefWatcher} has been constructed
 * with.
 *
 * @param referenceName An logical identifier for the watched object.
 */
public void watch(Object watchedReference, String referenceName) {
  if (this == DISABLED) {
    return;
  }
  checkNotNull(watchedReference, "watchedReference");
  checkNotNull(referenceName, "referenceName");
  final long watchStartNanoTime = System.nanoTime();
  String key = UUID.randomUUID().toString();
  retainedKeys.add(key);
  final KeyedWeakReference reference =
      new KeyedWeakReference(watchedReference, key, referenceName, queue);

  ensureGoneAsync(watchStartNanoTime, reference);
}
```

`watch()` 方法的参数是 `Object` ，`LeakCanary` 并不仅仅是针对 `Android` 的，它可以检测任何对象的内存泄漏，原理都是一致的。

这里出现了几个新面孔，先来了解一下各自是什么：

* `retainedKeys` : 一个 `Set<String>` 集合，每个检测的对象都对应着一个唯一的 `key`，存储在 `retainedKeys` 中
* `KeyedWeakReference` :  自定义的弱引用，持有检测对象和对用的 `key` 值

```Java
final class KeyedWeakReference extends WeakReference<Object> {
  public final String key;
  public final String name;

  KeyedWeakReference(Object referent, String key, String name,
      ReferenceQueue<Object> referenceQueue) {
    super(checkNotNull(referent, "referent"), checkNotNull(referenceQueue, "referenceQueue"));
    this.key = checkNotNull(key, "key");
    this.name = checkNotNull(name, "name");
  }
}
```

* `queue` : `ReferenceQueue` 对象，和 `KeyedWeakReference` 配合使用

这里有个小知识点，弱引用和引用队列 `ReferenceQueue` 联合使用时，如果弱引用持有的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中。即 `KeyedWeakReference` 持有的 `Activity` 对象如果被垃圾回收，该对象就会加入到引用队列 `queue` 中。

接着看看具体的内存泄漏判断过程：

### RefWatcher.ensureGoneAsync

```java
private void ensureGoneAsync(final long watchStartNanoTime, final KeyedWeakReference reference) {
  watchExecutor.execute(new Retryable() {
    @Override public Retryable.Result run() {
      return ensureGone(reference, watchStartNanoTime);
    }
  });
}
```
通过 `watchExecutor` 执行检测操作，这里的 `watchExecutor` 是 `AndroidWatchExecutor` 对象。

```Java
@Override protected WatchExecutor defaultWatchExecutor() {
  return new AndroidWatchExecutor(DEFAULT_WATCH_DELAY_MILLIS);
}
```
`DEFAULT_WATCH_DELAY_MILLIS` 为 5 s。

```Java
public AndroidWatchExecutor(long initialDelayMillis) {
  mainHandler = new Handler(Looper.getMainLooper());
  HandlerThread handlerThread = new HandlerThread(LEAK_CANARY_THREAD_NAME);
  handlerThread.start();
  backgroundHandler = new Handler(handlerThread.getLooper());
  this.initialDelayMillis = initialDelayMillis;
  maxBackoffFactor = Long.MAX_VALUE / initialDelayMillis;
}
```

看看其中用到的几个对象：

* `mainHandler` : 主线程消息队列
* `handlerThread` : 后台线程，`HandlerThread` 对象，线程名为 `LeakCanary-Heap-Dump`
* `backgroundHandler` : 上面的后台线程的消息队列
* `initialDelayMillis` : 5 s，即之前的 `DEFAULT_WATCH_DELAY_MILLIS`

```Java
@Override public void execute(Retryable retryable) {
  if (Looper.getMainLooper().getThread() == Thread.currentThread()) {
    waitForIdle(retryable, 0);
  } else {
    postWaitForIdle(retryable, 0);
  }
}

void postWaitForIdle(final Retryable retryable, final int failedAttempts) {
  mainHandler.post(new Runnable() {
    @Override public void run() {
      waitForIdle(retryable, failedAttempts);
    }
  });
}

void waitForIdle(final Retryable retryable, final int failedAttempts) {
  // This needs to be called from the main thread.
  Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
    @Override public boolean queueIdle() {
      postToBackgroundWithDelay(retryable, failedAttempts);
      return false;
    }
  });
}
```

在具体的 `execute()` 过程中，不管是 `waitForIdle` 还是 `postWaitForIdle`，最终还是要切换到主线程中执行。要注意的是，这里的 `IdleHandler` 到底是什么时候去执行？

我们都知道 `Handler` 是循环处理 `MessageQueue` 中的消息的，当消息队列中没有更多消息需要处理的时候，且声明了 `IdleHandler` 接口，这是就会去处理这里的操作。即指定一些操作，当线程空闲的时候来处理。当主线程空闲时，就会通知后台线程延时 5 秒执行内存泄漏检测工作。

```Java
void postToBackgroundWithDelay(final Retryable retryable, final int failedAttempts) {
  long exponentialBackoffFactor = (long) Math.min(Math.pow(2, failedAttempts), maxBackoffFactor);
  long delayMillis = initialDelayMillis * exponentialBackoffFactor;
  backgroundHandler.postDelayed(new Runnable() {
    @Override public void run() {
      Retryable.Result result = retryable.run();
      if (result == RETRY) {
        postWaitForIdle(retryable, failedAttempts + 1);
      }
    }
  }, delayMillis);
}
```

下面是真正的检测过程，`AndroidWatchExecutor` 在执行时调用 `ensureGone()` 方法：

### RefWatcher.ensureGone

```Java
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
  long gcStartNanoTime = System.nanoTime();
  long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);

  removeWeaklyReachableReferences();

  if (debuggerControl.isDebuggerAttached()) {
    // The debugger can create false leaks.
    return RETRY;
  }
  if (gone(reference)) {
    return DONE;
  }
  gcTrigger.runGc();
  removeWeaklyReachableReferences();
  if (!gone(reference)) {
    long startDumpHeap = System.nanoTime();
    long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);

    File heapDumpFile = heapDumper.dumpHeap();
    if (heapDumpFile == RETRY_LATER) {
      // Could not dump the heap.
      return RETRY;
    }
    long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
    heapdumpListener.analyze(
        new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
            gcDurationMs, heapDumpDurationMs));
  }
  return DONE;
}
```

再重复一次几个变量的含义，`retainedKeys` 是一个 `Set`集合，存储检测对象对应的唯一 `key` 值，`queue`是一个引用队列，存储被垃圾回收的对象。

主要过程有一下几步：

#### RefWatcher.emoveWeaklyReachableReferences()

```Java
private void removeWeaklyReachableReferences() {
  // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
  // reachable. This is before finalization or garbage collection has actually happened.
  KeyedWeakReference ref;
  while ((ref = (KeyedWeakReference) queue.poll()) != null) {
    retainedKeys.remove(ref.key);
  }
}
```

遍历引用队列 `queue`，判断队列中是否存在当前 `Activity` 的弱引用，存在则删除 `retainedKeys` 中对应的引用的 `key`值。

#### RefWatcher.gone()

```Java
private boolean gone(KeyedWeakReference reference) {
  return !retainedKeys.contains(reference.key);
}
```

判断 `retainedKeys` 中是否包含当前 `Activity` 引用的 `key` 值。

如果不包含，说明上一步操作中 `retainedKeys` 移除了该引用的 `key` 值，也就说上一步操作之前引用队列 `queue` 中包含该引用，`GC` 处理了该引用，未发生内存泄漏，返回 `DONE`，不再往下执行。

如果包含，并不会立即判定发生内存泄漏，可能存在某个对象已经不可达，但是尚未进入引用队列 `queue`。这时会主动执行一次 `GC` 操作之后再次进行判断。

#### gcTrigger.runGc()

```Java
/**
 * Called when a watched reference is expected to be weakly reachable, but hasn't been enqueued
 * in the reference queue yet. This gives the application a hook to run the GC before the {@link
 * RefWatcher} checks the reference queue again, to avoid taking a heap dump if possible.
 */
public interface GcTrigger {
  GcTrigger DEFAULT = new GcTrigger() {
    @Override public void runGc() {
      // Code taken from AOSP FinalizationTest:
      // https://android.googlesource.com/platform/libcore/+/master/support/src/test/java/libcore/
      // java/lang/ref/FinalizationTester.java
      // System.gc() does not garbage collect every time. Runtime.gc() is
      // more likely to perfom a gc.
      Runtime.getRuntime().gc();
      enqueueReferences();
      System.runFinalization();
    }

    private void enqueueReferences() {
      // Hack. We don't have a programmatic way to wait for the reference queue daemon to move
      // references to the appropriate queues.
      try {
        Thread.sleep(100);
      } catch (InterruptedException e) {
        throw new AssertionError();
      }
    }
  };

  void runGc();
}
```

注意这里调用 `GC` 的写法，并不是使用 `System.gc`。`System.gc` 仅仅只是通知系统在合适的时间进行一次垃圾回收操作，实际上并不能保证一定执行。

主动进行 `GC` 之后会再次进行判定，过程同上。首先调用 `removeWeaklyReachableReferences()` 清除 `retainedKeys` 中弱引用的 `key` 值，再判断是否移除。如果仍然没有移除，判定为内存泄漏。

## 内存泄露结果处理

### AndroidHeapDumper.dumpHeap

判定内存泄漏之后，调用 `heapDumper.dumpHeap()` 进行处理：

`AndroidHeapDumper.java`
```Java
@SuppressWarnings("ReferenceEquality") // Explicitly checking for named null.
@Override public File dumpHeap() {
  File heapDumpFile = leakDirectoryProvider.newHeapDumpFile();

  if (heapDumpFile == RETRY_LATER) {
    return RETRY_LATER;
  }

  FutureResult<Toast> waitingForToast = new FutureResult<>();
  showToast(waitingForToast);

  if (!waitingForToast.wait(5, SECONDS)) {
    CanaryLog.d("Did not dump heap, too much time waiting for Toast.");
    return RETRY_LATER;
  }

  Toast toast = waitingForToast.get();
  try {
    Debug.dumpHprofData(heapDumpFile.getAbsolutePath());
    cancelToast(toast);
    return heapDumpFile;
  } catch (Exception e) {
    CanaryLog.d(e, "Could not dump heap");
    // Abort heap dump
    return RETRY_LATER;
  }
}
```

`leakDirectoryProvider.newHeapDumpFile()` 新建了 `hprof` 文件，然后调用 `Debug.dumpHprofData()` 方法 `dump` 当前堆内存并写入刚才创建的文件。

回到 `RefWatcher.ensureGone()` 方法中，生成 `heapDumpFile` 文件之后，通过 `heapdumpListener` 分析。

### ServiceHeapDumpListener.analyze
```Java
heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
```

这里的 `heapdumpListener` 是 `ServiceHeapDumpListener` 对象，接着进入 `ServiceHeapDumpListener.runAnalysis()` 方法。

```Java
@Override public void analyze(HeapDump heapDump) {
  checkNotNull(heapDump, "heapDump");
  HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
}
```

这里的 `listenerServiceClass` 指的是 `DisplayLeakService.class`，文章开头提到的 `AndroidRefWatcherBuilder` 中进行了配置。

```Java
@Override protected HeapDump.Listener defaultHeapDumpListener() {
  return new ServiceHeapDumpListener(context, DisplayLeakService.class);
}
```
#### HeapAnalyzerService.runAnalysis
`HeapAnalyzerService.runAnalysis()` 方法中启动了它自己，传递了两个参数，`DisplayLeakService` 类名和要分析的 `heapDump`。启动自己后，在 `onHandleIntent` 中进行处理。

```Java
/**
 * This service runs in a separate process to avoid slowing down the app process or making it run
 * out of memory.
 */
public final class HeapAnalyzerService extends IntentService {

  private static final String LISTENER_CLASS_EXTRA = "listener_class_extra";
  private static final String HEAPDUMP_EXTRA = "heapdump_extra";

  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    context.startService(intent);
  }

  public HeapAnalyzerService() {
    super(HeapAnalyzerService.class.getSimpleName());
  }

  @Override protected void onHandleIntent(Intent intent) {
    if (intent == null) {
      CanaryLog.d("HeapAnalyzerService received a null intent, ignoring.");
      return;
    }
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    HeapAnalyzer heapAnalyzer = new HeapAnalyzer(heapDump.excludedRefs);

    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
}
```
#### heapAnalyzer.checkForLeak

`checkForLeak` 方法中主要使用了 `Square` 公司的另一个库 [haha](https://github.com/square/haha) 来分析 `Android heap dump`，得到结果后回调给 `DisplayLeakService`。

#### AbstractAnalysisResultService.sendResultToListener
```Java
public static void sendResultToListener(Context context, String listenerServiceClassName,
    HeapDump heapDump, AnalysisResult result) {
  Class<?> listenerServiceClass;
  try {
    listenerServiceClass = Class.forName(listenerServiceClassName);
  } catch (ClassNotFoundException e) {
    throw new RuntimeException(e);
  }
  Intent intent = new Intent(context, listenerServiceClass);
  intent.putExtra(HEAP_DUMP_EXTRA, heapDump);
  intent.putExtra(RESULT_EXTRA, result);
  context.startService(intent);
}
```

同样在 `onHandleIntent` 中进行处理。

#### DisplayLeakService.onHandleIntent
```Java
@Override protected final void onHandleIntent(Intent intent) {
  HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAP_DUMP_EXTRA);
  AnalysisResult result = (AnalysisResult) intent.getSerializableExtra(RESULT_EXTRA);
  try {
    onHeapAnalyzed(heapDump, result);
  } finally {
    //noinspection ResultOfMethodCallIgnored
    heapDump.heapDumpFile.delete();
  }
}
```
#### DisplayLeakService.onHeapAnalyzed

调用 `onHeapAnalyzed()` 之后，会将 `hprof` 文件删除。

`DisplayLeakService.java`
```Java
@Override protected final void onHeapAnalyzed(HeapDump heapDump, AnalysisResult result) {
  String leakInfo = leakInfo(this, heapDump, result, true);
  CanaryLog.d("%s", leakInfo);

  boolean resultSaved = false;
  boolean shouldSaveResult = result.leakFound || result.failure != null;
  if (shouldSaveResult) {
    heapDump = renameHeapdump(heapDump);
    resultSaved = saveResult(heapDump, result);
  }

  PendingIntent pendingIntent;
  String contentTitle;
  String contentText;

  if (!shouldSaveResult) {
    contentTitle = getString(R.string.leak_canary_no_leak_title);
    contentText = getString(R.string.leak_canary_no_leak_text);
    pendingIntent = null;
  } else if (resultSaved) {
    pendingIntent = DisplayLeakActivity.createPendingIntent(this, heapDump.referenceKey);

    if (result.failure == null) {
      String size = formatShortFileSize(this, result.retainedHeapSize);
      String className = classSimpleName(result.className);
      if (result.excludedLeak) {
        contentTitle = getString(R.string.leak_canary_leak_excluded, className, size);
      } else {
        contentTitle = getString(R.string.leak_canary_class_has_leaked, className, size);
      }
    } else {
      contentTitle = getString(R.string.leak_canary_analysis_failed);
    }
    contentText = getString(R.string.leak_canary_notification_message);
  } else {
    contentTitle = getString(R.string.leak_canary_could_not_save_title);
    contentText = getString(R.string.leak_canary_could_not_save_text);
    pendingIntent = null;
  }
  // New notification id every second.
  int notificationId = (int) (SystemClock.uptimeMillis() / 1000);
  showNotification(this, contentTitle, contentText, pendingIntent, notificationId);
  afterDefaultHandling(heapDump, result, leakInfo);
}
```

根据分析结果，调用 `showNotification()` 方法构建了一个 `Notification` 向开发者通知内存泄漏。

```Java
public static void showNotification(Context context, CharSequence contentTitle,
    CharSequence contentText, PendingIntent pendingIntent, int notificationId) {
  NotificationManager notificationManager =
      (NotificationManager) context.getSystemService(Context.NOTIFICATION_SERVICE);

  Notification notification;
  Notification.Builder builder = new Notification.Builder(context) //
      .setSmallIcon(R.drawable.leak_canary_notification)
      .setWhen(System.currentTimeMillis())
      .setContentTitle(contentTitle)
      .setContentText(contentText)
      .setAutoCancel(true)
      .setContentIntent(pendingIntent);
  if (SDK_INT >= O) {
    String channelName = context.getString(R.string.leak_canary_notification_channel);
    setupNotificationChannel(channelName, notificationManager, builder);
  }
  if (SDK_INT < JELLY_BEAN) {
    notification = builder.getNotification();
  } else {
    notification = builder.build();
  }
  notificationManager.notify(notificationId, notification);
}
```

#### DisplayLeakService.afterDefaultHandling

最后还会执行一个空实现的方法 `afterDefaultHandling`：

```Java
/**
 * You can override this method and do a blocking call to a server to upload the leak trace and
 * the heap dump. Don't forget to check {@link AnalysisResult#leakFound} and {@link
 * AnalysisResult#excludedLeak} first.
 */
protected void afterDefaultHandling(HeapDump heapDump, AnalysisResult result, String leakInfo) {
}
```

你可以重写这个方法进行一些自定义的操作，比如向服务器上传泄漏的堆栈信息等。

这样，`LeakCanary` 就完成了整个内存泄漏检测的过程。可以看到，`LeakCanary` 的设计思路十分巧妙，同时也很清晰，有很多有意思的知识点，像对于弱引用和 `ReferenceQueue` 的使用， `IdleHandler` 的使用，四大组件的开启和关闭等等，都很值的大家去深究。

> 文章同步更新于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！

![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
