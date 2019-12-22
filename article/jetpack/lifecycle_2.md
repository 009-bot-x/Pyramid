# 硬核讲解 Jetpack 之 LifeCycle 源码篇

前一篇 [硬核讲解 Jetpack 之 LifeCycle 使用篇](https://juejin.im/post/5df64c19518825121d6e2013#heading-0) 主要介绍了 LifeCycle 存在的意义，基本和进阶的使用方法。今天话不多说，直接开始撸源码。

本文基于我手里的 [android_9.0.0_r45](https://github.com/lulululbj/android_9.0.0_r45) 源码，所有相关源码包括注释都上传到了我的 Github ，可以直接 clone 下来对照文章查看。

## LifeCycle 三剑客

在正式阅读源码之前，很有必要先介绍几个名词，**LifeCycleOwner** ，**LifecycleObserver**，**LifeCycle** 。

`LifeCycleOwner` 是一个接口 , 接口通常用来声明具备某种能力。`LifeCycleOwner` 的能力就是具备生命周期。典型的生命周期组件有 `Activity` 和 `Fragment` 。当然，我们也可以自定义生命周期组件。`LifeCycleOwner` 提供了 `getLifeCycle()` 方法来获取其 `LifeCycle` 对象。

```java
public interface LifecycleOwner {

    @NonNull
    Lifecycle getLifecycle();
}
```

`LifeCycleObserver` 是生命周期观察者，它是一个空接口。它没有任何方法，而是通过依赖 `OnLifecycleEvent` 注解来回调生命周期。

```java
public interface LifecycleObserver {

}
```

**生命周期组件** 和 **生命周期观察者** 都有了，`LifeCycle` 就是它们之间的桥梁。

`LifeCycle` 是具体的生命周期对象，每个 `LifeCycleOwner` 都会持有 `LifeCycle` 。通过 LifeCycle 我们可以获取当前生命周期状态，添加/删除 生命周期观察者。

`LifeCycle` 内部定义了两个枚举类，`Event` 和 `State` 。`Event` 表示生命周期事件，与 LifeCycleOwner 的生命周期事件是相对应的。

```java
public enum Event {
    /**
     * Constant for onCreate event of the {@link LifecycleOwner}.
     */
    ON_CREATE,
    /**
     * Constant for onStart event of the {@link LifecycleOwner}.
     */
    ON_START,
    /**
     * Constant for onResume event of the {@link LifecycleOwner}.
     */
    ON_RESUME,
    /**
     * Constant for onPause event of the {@link LifecycleOwner}.
     */
    ON_PAUSE,
    /**
     * Constant for onStop event of the {@link LifecycleOwner}.
     */
    ON_STOP,
    /**
     * Constant for onDestroy event of the {@link LifecycleOwner}.
     */
    ON_DESTROY,
    /**
     * An {@link Event Event} constant that can be used to match all events.
     */
    ON_ANY
}
```

`ON_ANY` 比较特殊，它表示任意生命周期事件。为什么要设计 `ON_ANY` 呢？其实我也不知道，暂时还没发现它的用处。

另一个枚举类 `State` 表示生命周期状态。

```java
public enum State {
        /**
         * 在此之后，Lifecycle 不会再派发生命周期事件。
         * 此状态在 Activity.onDestroy() 之前
         */
        DESTROYED,

        /**
         * 在 Activity 已经实例化但未 onCreate() 之前
         */
        INITIALIZED,

        /**
         * 在 Activity 的 onCreate() 之后到 onStop() 之前
         */
        CREATED,

        /**
         * 在 Activity 的 onStart() 之后到 onPause() 之前
         */
        STARTED,

        /**
         * 在 Activity 的 onResume() 之后
         */
        RESUMED;

        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
```

 `State` 可能相对比较难以理解，特别是其中枚举值的顺序。这里先不详细解读，但是务必记住源码中定义的枚举值顺序，`DESTROYED —— INITIALIZED —— CREATED —— STARTED ——RESUMED`，这个对于后面源码的理解特别重要。

梳理一下三剑客的关系。生命周期组件 `LifeCycleOwner` 在进入特定的生命周期后，发送特定的生命周期事件 `Event` ，通知 `LIfeCycle` 进入特定的 `State` ，进而回调生命周期观察者 `LifeCycleObserver` 的指定方法。

## 从 addObserver() 下手

面对源码无从下手的话，我们就从 LifeCycle 的基本使用入手。

```java
lifecycle.addObserver(LocationUtil( ))
```

`lifecycle` 其实就是 `getLifeCycle()`方法，只是在 Kotlin中被 简写了。`getLifeCycle()` 是接口 `LifeCycleOwner` 的方法。而 `AppCompatActivity` 并没有直接实现 LifeCycleOwner，它的父类 `FragmentActivity` 也没有，在它的爷爷类 `ComponentActivity` 中才找到 LifeCycleOwner 的踪影，看一下接口的实现。

```java
@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```

`mLifecycleRegistry` 是 `LifecycleRegistry` 对象，`LifecycleRegistry` 是 `LifeCycle` 的实现类。那么这里的 `LifecycleRegistry` 就是我们的生命周期对象了。来看一下它的 `addObserver()` 方法。

```java
......

// 保存 LifecycleObserver 及其对应的 State
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
        new FastSafeIterableMap<>();

 // 当前生命周期状态
private State mState;

/**
 * 添加生命周期观察者 LifecycleObserver
 * 另外要注意生命周期事件的 “倒灌”，如果在 onResume() 中调用 addObserver()，
 * 那么，观察者依然可以接收到 onCreate 和 onStart 事件。
 * 这么做的目的是保证 mObserverMap 中的 LifecycleObserver 始终保持在同一状态
 */
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    // ObserverWithState 是一个静态内部类
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    // 判断是否重入
    boolean isReentrance = mAddingObserverCounter != 0 || mHandlinengEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;

    // 如果观察者的初始状态小于 targetState ，则同步到 targetState
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

这里面要注意两个问题。第一个问题是生命周期的 "倒灌问题" ，这是我从 LiveData 那里借来的一次词。具体是什么问题呢？来举一个具体的例子，即使你在 `onResume( )` 中调用 `addObserver( )` 方法，观察者依然可以依次接收到 `onCreate` 和 `onStart` 事件 ，最终同步到 `targetState` 。这个 targetState 是通过 `calculateTargetState(observer)` 方法计算处理的。

```java
/**
 * 计算出的 targetState 一定是小于等于当前 mState 的
 */
  private State calculateTargetState(LifecycleObserver observer) {
    // 获取当前 Observer 的前一个 Observer
      Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);

      State siblingState = previous != null ? previous.getValue().mState : null;
  // 无重入情况下可不考虑 parentState ，为 null
      State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
              : null;
      return min(min(mState, siblingState), parentState);
  }
```

我们可以添加多个生命周期观察者，这时候就得注意维护它们的状态。每次添加的新的观察者的初始状态是 `INITIALIZED` ，需要把它同步到当前生命周期状态，确切的说，同步到一个不大于当前状态的 `targetState` 。从源码中的计算方式也有所体现，取得是 当前状态 mState，mObserverMap 中最后一个观察者的状态，有重入情况下 parentState 的状态 这三者中的最小值。

为什么要取这个最小值呢？我是这么理解的，当有新的生命周期事件时，需要将 `mObserverMap` 中的所有观察者都同步到新的同一状态，这个同步过程可能尚未完成，所以新加入的观察者只能先同步到最小状态。在新的观察者每改变一次生命周期，都会调用 `calculateTargetState()` 重新计算 `targetState` 。

最终的稳定状态下，没有生命周期切换，没有添加新的观察者，`mObserverMap` 中的所有观察者应该处于同一个生命周期状态。

## 谁来分发生命周期事件？

观察者已经添加完成了，那么如何将生命周期的变化通知观察者呢？

再回到 `ComponentActivity` ，你会发现里面并没有重写所有的生命周期函数。唯一让人可疑的就只有 `onCreate()` 当中的一行代码。

```java
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mSavedStateRegistryController.performRestore(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
    if (mContentLayoutId != 0) {
        setContentView(mContentLayoutId);
    }
}
```

这里的 `ReportFragment` 就是问题的答案。追进 `injectIfNeededIn()` 方法。

```java
public static void injectIfNeededIn(Activity activity) {
    // 使用 android.app.FragmentManager 保持兼容
    android.app.FragmentManager manager = activity.getFragmentManager();
    if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
        manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
        // Hopefully, we are the first to make a transaction.
        manager.executePendingTransactions();
    }
}
```

这里向 Activity 注入了一个没有页面的 Fragment 。这就让我想到了一些动态权限库也是这个套路，通过注入 Fragment 来代理权限请求。不出意外，`ReportFragment` 才是真正分发生命周期的地方。

```java
@Override
 public void onActivityCreated(Bundle savedInstanceState) {
     super.onActivityCreated(savedInstanceState);
     dispatchCreate(mProcessListener);
     dispatch(Lifecycle.Event.ON_CREATE);
 }

 @Override
 public void onStart() {
     super.onStart();
     dispatchStart(mProcessListener);
     dispatch(Lifecycle.Event.ON_START);
 }

 @Override
 public void onResume() {
     super.onResume();
     dispatchResume(mProcessListener);
     dispatch(Lifecycle.Event.ON_RESUME);
 }

 @Override
 public void onPause() {
     super.onPause();
     dispatch(Lifecycle.Event.ON_PAUSE);
 }

 @Override
 public void onStop() {
     super.onStop();
     dispatch(Lifecycle.Event.ON_STOP);
 }

 @Override
 public void onDestroy() {
     super.onDestroy();
     dispatch(Lifecycle.Event.ON_DESTROY);
     // just want to be sure that we won't leak reference to an activity
     mProcessListener = null;
 }
```

`mProcessListener` 是处理应用进程生命周期的，暂时不去管它。先看一下 `dispatch()` 方法。

```java
private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            // 调用 LifecycleRegistry.handleLifecycleEvent() 方法
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

在 Activity 中注入 `ReportFragment` ，在 ReportFragment 的各个生命周期函数中通过 `dispatch()` 方法来分发生命周期事件。 最后是通过  `LifecycleRegistry` 的 `handleLifecycleEvent()` 方法来处理 。为了方便后面的代码理解，这里假定 `handleLifecycleEvent()` 方法中的参数是  `ON_RESUME` ，即现在要经历从 `onStart()` 同步到 `onResume()` 的过程 。

```java
// 设置当前状态并通知观察者
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```

`getStateAfter()` 的作用是根据 Event 获取特定的 State ，并通知观察者同步到此生命周期状态。

```java
static State getStateAfter(Event event) {
    switch (event) {
        case ON_CREATE:
        case ON_STOP:
            return CREATED;
        case ON_START:
        case ON_PAUSE:
            return STARTED;
        case ON_RESUME:
            return RESUMED;
        case ON_DESTROY:
            return DESTROYED;
        case ON_ANY:
            break;
    }
    throw new IllegalArgumentException("Unexpected event value " + event);
}
```

参数是 `ON_RESUME` ，所以需要同步到的状态是 `RESUMED` 。接下来看看 `moveToState()` 方法的逻辑。

```java
private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}
```

首先将要同步到的生命周期状态赋给 `zhuangtaimState` ，此时 `mState` 的值就是 `RESUMED` 。然后调用 `sync()` 方法同步所有观察者的状态。

```java
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                + "new events from it.");
        return;
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // mState 是当前状态，如果 mState 小于 mObserverMap 中的状态值，调用 backwardPass()
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        // 如果 mState 大于 mObserverMap 中的状态值，调用 forwardPass()
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

这里会比较 `mState` 和  `mObserverMap` 中观察者的 State 值，判断是需要向前还是向后同步状态。现在 `mState` 的值是 `RESUMED` , 而观察者还停留在上一状态 `STARTED` ，所以观察者的状态都得往前挪一步，这里调用的是 `forwardPass()` 方法。

```java
private void forwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
            mObserverMap.iteratorWithAdditions();
    while (ascendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
        ObserverWithState observer = entry.getValue();
        // 向上传递事件，直到 observer 的状态值等于当前状态值
        while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            pushParentState(observer.mState);
            // 分发生命周期事件
            observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
            popParentState();
        }
    }
}
```

最终会调用 `ObserverWithState` 的 `dispatchEvent()` 方法。

这里先暂停一下，不继续往下追源码。上面假定的场景是 `ON_START` 到 `ON_RESUME` 的过程。现在假定另一个场景，
`ON_RESUME` 到 `ON_PAUSE` ，流程如下。

1.  `handleLifecycleEvent()` 方法的参数是 `ON_PAUSE`
2.  `getStateAfter()` 得到要同步到的状态是  `STARTED` ，并赋给 `mState`，接着调用 `moveToState()`
3. `moveToState(STARTED)` 中调用 `sync()` 方法同步
4. `sync()` 方法中，`mState` 的值是 `STARTED` ，而 `mObserverMap` 中观察者的状态都是 `RESUMED` 。所以观察者们都需要往后挪一步，这调用的就是 `backwardPass()` 方法。

`backwardPass()` 方法其实和 `forwardPass()` 差不多。

```java
private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        // 向下传递事件，直到 observer 的状态值等于当前状态值
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            // 分发生命周期事件
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
```

二者唯一的区别就是获取要分发的事件，一个是 `upEvent()` ，一个是 `downEvent()` 。

`upEvent() ` 是获取 state 升级所需要经历的事件，`downEvent()` 是获取 state 降级所需要经历的事件。其实从源码中一目了然。

```java
private static Event upEvent(State state) {
    switch (state) {
        case INITIALIZED:
        case DESTROYED:
            return ON_CREATE;
        case CREATED:
            return ON_START;
        case STARTED:
            return ON_RESUME;
        case RESUMED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}

private static Event downEvent(State state) {
    switch (state) {
        case INITIALIZED:
            throw new IllegalArgumentException();
        case CREATED:
            return ON_DESTROY;
        case STARTED:
            return ON_STOP;
        case RESUMED:
            return ON_PAUSE;
        case DESTROYED:
            throw new IllegalArgumentException();
    }
    throw new IllegalArgumentException("Unexpected state value " + state);
}
```

从 `STARTED` 到 `RESUMED` 需要升级，`upEvent(STARTED)` 的返回值是 `ON_RESUME` 。
从 `RESUMED` 到 `STARTED` 需要降级，`downEvent(RESUMED)`的返回值是 `ON_PAUSE` 。

看到这不知道你有没有一点懵，State 和 Event  的关系我也摸索了很长一段时间才理清楚。首先还记得 `State` 的枚举值顺序吗？

```java
DESTROYED —— INITIALIZED —— CREATED —— STARTED —— RESUMED
```

`DESTROYED` 最小，`RESUMED` 最大 。仔细看一下这几个枚举值，你会发现并没有 `PAUSED` 。`onResume` 进入到 `onPause` 阶段最后分发的的确是 `ON_PAUSE` ，但是将观察者的状态置为了 `STARTED` 。

关于 `State` 和 `Event` 的关系，官网给出了一张图，如下所所示：

![](https://developer.android.google.cn/images/topic/libraries/architecture/lifecycle-states.svg)

但我不得不说，画的的确有点抽象，其实应该换个画法。再来一张我在 [这里](https://www.jianshu.com/p/1d2d566e5690) 看到的一张图：

![](https://upload-images.jianshu.io/upload_images/2669479-df0bb30ab769a55e.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

**状态之间的事件**，**事件之后的状态**，**状态之间的大小** ，是不是有种一目了然的感觉？理解这种图很重要，可以说搞不清 Event 和 State 的关系，就看不懂这部分的源码。

再回到之前的源码解析，同步 Observer 生命周期的 `sync()` 方法最终会调用 `ObserverWithState` 的 `dispatchEvent()` 方法。

```java
static class ObserverWithState {
    State mState;
    GenericLifecycleObserver mLifecycleObserver;

    ObserverWithState(LifecycleObserver observer, State initialState) {
        mLifecycleObserver = Lifecycling.getCallback(observer);
        mState = initialState;
    }

    void dispatchEvent(LifecycleOwner owner, Event event) {
        State newState = getStateAfter(event);
        mState = min(mState, newState);
        // ReflectiveGenericLifecycleObserver.onStateChanged()
        mLifecycleObserver.onStateChanged(owner, event);
        mState = newState;
    }
}
```

`mLifecycleObserver` 通过 `Lifecycling.getCallback()` 方法赋值。

```java
@NonNull
static GenericLifecycleObserver getCallback(Object object) {
    if (object instanceof FullLifecycleObserver) {
        return new FullLifecycleObserverAdapter((FullLifecycleObserver) object);
    }

    if (object instanceof GenericLifecycleObserver) {
        return (GenericLifecycleObserver) object;
    }

    final Class<?> klass = object.getClass();
    int type = getObserverConstructorType(klass);
    // 获取 type
    // GENERATED_CALLBACK 表示注解生成的代码
    // REFLECTIVE_CALLBACK 表示使用反射
    if (type == GENERATED_CALLBACK) {
        List<Constructor<? extends GeneratedAdapter>> constructors =
                sClassToAdapters.get(klass);
        if (constructors.size() == 1) {
            GeneratedAdapter generatedAdapter = createGeneratedAdapter(
                    constructors.get(0), object);
            return new SingleGeneratedAdapterObserver(generatedAdapter);
        }
        GeneratedAdapter[] adapters = new GeneratedAdapter[constructors.size()];
        for (int i = 0; i < constructors.size(); i++) {
            adapters[i] = createGeneratedAdapter(constructors.get(i), object);
        }
        return new CompositeGeneratedAdaptersObserver(adapters);
    }
    return new ReflectiveGenericLifecycleObserver(object);
}
```

如果使用的是 `DefaultLifecycleObserver` ，而 `DefaultLifecycleObserver` 又是继承 `FullLifecycleObserver` 的，所以这里会返回 `FullLifecycleObserverAdapter` 。

如果只是普通的 `LifecycleObserver` ，那么就需要通过 `getObserverConstructorType()` 方法判断使用的是注解还是反射。

```java
private static int getObserverConstructorType(Class<?> klass) {
    if (sCallbackCache.containsKey(klass)) {
        return sCallbackCache.get(klass);
    }
    int type = resolveObserverCallbackType(klass);
    sCallbackCache.put(klass, type);
    return type;
}

private static int resolveObserverCallbackType(Class<?> klass) {
    // anonymous class bug:35073837
    // 匿名内部类使用反射
    if (klass.getCanonicalName() == null) {
        return REFLECTIVE_CALLBACK;
    }

    // 寻找注解生成的 GeneratedAdapter 类
    Constructor<? extends GeneratedAdapter> constructor = generatedConstructor(klass);
    if (constructor != null) {
        sClassToAdapters.put(klass, Collections
                .<Constructor<? extends GeneratedAdapter>>singletonList(constructor));
        return GENERATED_CALLBACK;
    }

    // 寻找被 OnLifecycleEvent 注解的方法
    boolean hasLifecycleMethods = ClassesInfoCache.sInstance.hasLifecycleMethods(klass);
    if (hasLifecycleMethods) {
        return REFLECTIVE_CALLBACK;
    }

    // 没有找到注解生成的 GeneratedAdapter 类，也没有找到 OnLifecycleEvent 注解，
    // 则向上寻找父类
    Class<?> superclass = klass.getSuperclass();
    List<Constructor<? extends GeneratedAdapter>> adapterConstructors = null;
    if (isLifecycleParent(superclass)) {
        if (getObserverConstructorType(superclass) == REFLECTIVE_CALLBACK) {
            return REFLECTIVE_CALLBACK;
        }
        adapterConstructors = new ArrayList<>(sClassToAdapters.get(superclass));
    }

    // 寻找是否有接口实现
    for (Class<?> intrface : klass.getInterfaces()) {
        if (!isLifecycleParent(intrface)) {
            continue;
        }
        if (getObserverConstructorType(intrface) == REFLECTIVE_CALLBACK) {
            return REFLECTIVE_CALLBACK;
        }
        if (adapterConstructors == null) {
            adapterConstructors = new ArrayList<>();
        }
        adapterConstructors.addAll(sClassToAdapters.get(intrface));
    }
    if (adapterConstructors != null) {
        sClassToAdapters.put(klass, adapterConstructors);
        return GENERATED_CALLBACK;
    }

    return REFLECTIVE_CALLBACK;
}
```

在普通的 Activity 中注册观察者调用的是 `ReflectiveGenericLifecycleObserver.onStateChanged()`  。

```java
class ReflectiveGenericLifecycleObserver implements GenericLifecycleObserver {
    private final Object mWrapped; // Observer 对象
    private final CallbackInfo mInfo; // 反射获取注解信息

    ReflectiveGenericLifecycleObserver(Object wrapped) {
        mWrapped = wrapped;
        mInfo = ClassesInfoCache.sInstance.getInfo(mWrapped.getClass());
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Event event) {
        // 调用 ClassesInfoCache.CallbackInfo.invokeCallbacks()
        mInfo.invokeCallbacks(source, event, mWrapped);
    }
}
```

再追进 `ClassesInfoCache.CallbackInfo.invokeCallbacks()` 方法。

```java
void invokeCallbacks(LifecycleOwner source, Lifecycle.Event event, Object target) {
    // 不仅分发了当前生命周期事件，还分发了 ON_ANY
    invokeMethodsForEvent(mEventToHandlers.get(event), source, event, target);
    invokeMethodsForEvent(mEventToHandlers.get(Lifecycle.Event.ON_ANY), source, event,
            target);
}

private static void invokeMethodsForEvent(List<MethodReference> handlers,
        LifecycleOwner source, Lifecycle.Event event, Object mWrapped) {
    if (handlers != null) {
        for (int i = handlers.size() - 1; i >= 0; i--) {
            handlers.get(i).invokeCallback(source, event, mWrapped);
        }
    }
}

void invokeCallback(LifecycleOwner source, Lifecycle.Event event, Object target) {
    //noinspection TryWithIdenticalCatches
    try {
        switch (mCallType) {
            case CALL_TYPE_NO_ARG:
                mMethod.invoke(target);
                break;
            case CALL_TYPE_PROVIDER:
                mMethod.invoke(target, source);
                break;
            case CALL_TYPE_PROVIDER_WITH_EVENT:
                mMethod.invoke(target, source, event);
                break;
        }
    } catch (InvocationTargetException e) {
        throw new RuntimeException("Failed to call observer method", e.getCause());
    } catch (IllegalAccessException e) {
        throw new RuntimeException(e);
    }
}
```

最后反射调用注解标记的生命周期方法。

## 总结
