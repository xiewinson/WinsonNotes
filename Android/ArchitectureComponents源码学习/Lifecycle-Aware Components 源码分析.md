# Lifecycle-Aware Components 源码分析

> 去年谷歌推出了 Architecture Components，其中 Lifecycle 组件可以说是重中之重，也是 LiveData 和 ViewModel 两个组件的重要基础。
## 使用方式
Lifecycle 的使用倒是特别的简单，首先声明一个实现了  `LifecycleObserver` 接口的类
```Java
// 新建一个类实现 LifecycleObserver 接口
public class MyComponent implements LifecycleObserver{

    // 需要依赖某个生命周期的类需要注解显示出相应的生命周期
    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    public void onAttach(){
        LogUtil.d("MyComponent->onAttach");
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    public void onDetach(){
        LogUtil.d("MyComponent->onDetach");
    }
}    
```
然后在 support 包中的 Activity/Fragment 中使用(后面讲为啥是 support 中的)
```Java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        
        // 加入观察者中
        getLifecycle().addObserver(new MyComponent());
    }
    
}

```
## 源码分析
首先明确下这个东西的三个角色：
1. __LifecycleRegistry__ 注册 Lifecycle 的东西，一般是注册在 Activity、Fragmnet、Service 中，support library 中的 Activity 和 Fragment 中就自带了
2. __LifecycleOwner__ LifecycleRegistry 的构造器就需要这个东西，这是 lifecycle 的提供者，自然就是 Activity/Fragment/Service 本身了
3. __LifecyclerObserver__ 监听生命周期的自定义组件

现在去源码里来找到 `getLifecycle()` 的相关信息，一路走下来发现 `AppCompatActivity` 的父类一直往上寻找，有一个 `SupportActivity`，`getLifecycle()` 方法就属于它，这个方法返回一个全局变量 `mLifecycleRegistry`，下面的代码只贴出了这个类的关键方法
```Java
public class SupportActivity extends Activity implements LifecycleOwner {


    private LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

    @Override
    @SuppressWarnings("RestrictedApi")
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ReportFragment.injectIfNeededIn(this);
    }

    @CallSuper
    @Override
    protected void onSaveInstanceState(Bundle outState) {
        mLifecycleRegistry.markState(Lifecycle.State.CREATED);
        super.onSaveInstanceState(outState);
    }

    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }

}

```
从代码看出来 `SupportActivity` 实现了 `LifecycleOwner` 接口，`getLifecycle` 就是 `LifecycleOwner` 的声明的方法。这代码看起来平平无奇，怎么就能监听生命周期了？先放下 `LifecycleRegistry` ，看看 `SupportActivity` 的 `onCreate `中所做的工作。`ReportFragment.injectIfNeededIn(this)` 看起来是插入了一个 `Fragment`，代码很少，下面看看它的代码
```Java
public class ReportFragment extends Fragment {
    private static final String REPORT_FRAGMENT_TAG = "android.arch.lifecycle"
            + ".LifecycleDispatcher.report_fragment_tag";

    public static void injectIfNeededIn(Activity activity) {
        // ProcessLifecycleOwner should always correctly work and some activities may not extend
        // FragmentActivity from support lib, so we use framework fragments for activities
        android.app.FragmentManager manager = activity.getFragmentManager();
        if (manager.findFragmentByTag(REPORT_FRAGMENT_TAG) == null) {
            manager.beginTransaction().add(new ReportFragment(), REPORT_FRAGMENT_TAG).commit();
            // Hopefully, we are the first to make a transaction.
            manager.executePendingTransactions();
        }
    }

    static ReportFragment get(Activity activity) {
        return (ReportFragment) activity.getFragmentManager().findFragmentByTag(
                REPORT_FRAGMENT_TAG);
    }

    private ActivityInitializationListener mProcessListener;

    private void dispatchCreate(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onCreate();
        }
    }

    private void dispatchStart(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onStart();
        }
    }

    private void dispatchResume(ActivityInitializationListener listener) {
        if (listener != null) {
            listener.onResume();
        }
    }

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

    private void dispatch(Lifecycle.Event event) {
        Activity activity = getActivity();
        if (activity instanceof LifecycleRegistryOwner) {
            ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
            return;
        }

        if (activity instanceof LifecycleOwner) {
            Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
            if (lifecycle instanceof LifecycleRegistry) {
                ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
            }
        }
    }

    void setProcessListener(ActivityInitializationListener processListener) {
        mProcessListener = processListener;
    }

    interface ActivityInitializationListener {
        void onCreate();

        void onStart();

        void onResume();
    }
}

```
好的，这个代码很清晰明了：
* `injectIfNeededIn` 方法判断了该 `Fragment` 是否已存在于 `Activity` 中了，如果没有就添加进去，并且这是一个没有界面的 `Fragment`
* `setProcessListener` 方法添加了一个 `ActivityInitializationListener`，在 `onActivityCreated`、 `onStart`、`onResume` 这几个声明周期会去回调一下相应生命周期回调方法
* 该 `Fragment` 的在执行生命周期回调方法的时候会调用 `dispatch` 方法，该方法的参数是一个枚举，枚举值列举了主要生命周期。然后这个方法中判断了 `Fragment` 所属的 `Activity` ，最终都会走到获取到 `LifecycleRegistry`，然后调用它的 `handleLifecycleEvent` 去处理。这时候回想上文，`SupportActivity` 不就是实现了 `LifecycleOwner` 接口，并且它的 `getLifecycle` 返回的也正好是个 `LifecycleRegistry` 对象

从这里我们就大概猜到了端倪，`SupportActivity` 中插入了一个 `ReportFragment` 用来监听 `Activity` 的生命周期，然后在合适的时候回调相应的方法。这时候先打断一下，我们目前分析的 `SupportActivity` 的实现而不是 `Activity` 的情况，`SupportActivity` 是一个 `LifecycleOwner`，后面再分析 `LifecycleRegistry` 的情况。好的继续，现在应该分析 `LifecycleRegistry` 的源码了。

首先，这个类的注释先说明了 `It is used by Fragments and Support Library Activities`，人家是用在 `Fragment` 和 `Support Library Activity` 的。这个类继承于 `Lifecycle` 这个 `abstract` 类，`Lifecycle` 的代码非常简单，直接贴上来吧。
```Java
public abstract class Lifecycle {

    @MainThread
    public abstract void addObserver(@NonNull LifecycleObserver observer);

    @MainThread
    public abstract void removeObserver(@NonNull LifecycleObserver observer);
    
    @MainThread
    @NonNull
    public abstract State getCurrentState();

    @SuppressWarnings("WeakerAccess")
    public enum Event {
        ON_CREATE,
        ON_START,
        ON_RESUME,
        ON_PAUSE,
        ON_STOP,
        ON_DESTROY,
        ON_ANY
    }

    @SuppressWarnings("WeakerAccess")
    public enum State {
        DESTROYED,
        INITIALIZED,
        CREATED,
        STARTED,
        RESUMED;

        public boolean isAtLeast(@NonNull State state) {
            return compareTo(state) >= 0;
        }
    }
}

```
这个 `addObserver` 很眼熟，不就是刚才写 Demo 的时候调用的吗？好的，可以去 `LifecycleRegistry` 的源码看看了。

首先它的构造器需要传入一个 `LifecycleOwner`，恩，上文提过了，`SupportAcitivity` 就实现了这玩意儿所以在那就直接传了个 `this`，然后看到这里拿了个弱引用来存，应该这样的，毕竟持有的是个一不小心就引起内存泄漏的玩意儿。这里还有个 `mState` 被设为 `INITIALIZED` 初始化状态

```Java
public LifecycleRegistry(@NonNull LifecycleOwner provider) {
        mLifecycleOwner = new WeakReference<>(provider);
        mState = INITIALIZED;
}
```
接下来看看 `addObserver` 干了啥，代码贴在下方。这里的 `mObserverMap` 是一个 `FastSafeIterableMap`，按照注释所说的这是一个支持在遍历的时候修改的 `Map`，与主题无关先不深究。方法参数的 `observer` 被包装成一个 `ObserverWithState` 然后放置到 `mObserverMap` 中。并且，在这一步的操作中，会进行同步一下生命周期，比如你自己声明的组件中有方法是注解的 `onCreate` 时候执行，但你在 `onResume` 才 `addObserver`，那这时候也会去执行那个注解了 `onCreate` 时执行的方法
```Java
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
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

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
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
值得注意的一个方法是 `handleLifecycleEvent`，这个方法就是上文提到的 `ReportFragment` 在执行生命周期回调方法时调用的方法，来看看它的代码，很简单
```Java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}
```
`getStateAfter` 方法通过当前的生命周期 Event 枚举值去取得下一个状态的 State 枚举值。取得 State 之后，使用 `moveToState(State next)` 方法去通知改变状态，里面主要是调用了 `sync()` 方法。
```Java
private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            Log.w(LOG_TAG, "LifecycleOwner is garbage collected, you shouldn't try dispatch "
                    + "new events from it.");
            return;
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
```
这里面会循环处理和改变 `mObserverMap` 的状态，如果当前生命周期早于 observer 所处的生命周期，那么使用 `backwardPass` 方法，否则调用 `forwardPass`
```Java
private void forwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> ascendingIterator =
                mObserverMap.iteratorWithAdditions();
        while (ascendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = ascendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) < 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                pushParentState(observer.mState);
                observer.dispatchEvent(lifecycleOwner, upEvent(observer.mState));
                popParentState();
            }
        }
    }

    private void backwardPass(LifecycleOwner lifecycleOwner) {
        Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
                mObserverMap.descendingIterator();
        while (descendingIterator.hasNext() && !mNewEventOccurred) {
            Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
            ObserverWithState observer = entry.getValue();
            while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                    && mObserverMap.contains(entry.getKey()))) {
                Event event = downEvent(observer.mState);
                pushParentState(getStateAfter(event));
                observer.dispatchEvent(lifecycleOwner, event);
                popParentState();
            }
        }
    }
```
这两个方法分别使用了 `downEvent` 和 `upEvent` 方法来把 State 转换为 Event，规则如图
![image](https://developer.android.com/images/topic/libraries/architecture/lifecycle-states.png)
为啥会发生生命周期退后的情况呢？想想也对，比如 Activity1 打开了新 Activity2 ，那么 Activity1 生命周期也经历了 `onPause` 和 `onStop`，所以对应的 State 也应该从 `RESUMED` 退回到 `CREATED`，然后返回 Activity1 时，生命周期要走 `onStart` 和 `onResume`，所以 State 要走 `STARTED` 和 `RESUMED`

好了，这部分其实没什么好说的了，回过头说说刚才 `addObserver` 时候添加到 Map 中的 `ObserverWithState`，先看源码
```Java
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
            mLifecycleObserver.onStateChanged(owner, event);
            mState = newState;
        }
    }
```
刚才说到的生命周期改变都会调用到这个 observer 的 `dispatchEvent` 方法，里面其实能猜得到就是去调用你声明的注解了声明周期的方法。`Lifecycling.getCallback(observer)` 这个方法会去构建一个 `GenericLifecycleObserver`。这时候回过头想想你在自定义组件的那个注解，`OnLifecycleEvent`，打开源码一看原来是个 runtime 的，恩，所以它是用来反射取得这些方法的。但是，如果你在 gradle 中引入 `annotationProcessor "android.arch.lifecycle:compiler:1.1.1"`，那么编译时会根据你的注解生成一个后缀名为 `类名_LifecycleAdapter` 的类，来处理生命周期事件。很明显，这种方式优雅得多。下面就是我上面声明的 `MyComponent` 生成的类的代码
```Java
public class MyComponent_LifecycleAdapter implements GeneratedAdapter {
  final MyComponent mReceiver;

  MyComponent_LifecycleAdapter(MyComponent receiver) {
    this.mReceiver = receiver;
  }

  @Override
  public void callMethods(LifecycleOwner owner, Lifecycle.Event event, boolean onAny,
      MethodCallsLogger logger) {
    boolean hasLogger = logger != null;
    if (onAny) {
      return;
    }
    if (event == Lifecycle.Event.ON_RESUME) {
      if (!hasLogger || logger.approveCall("onAttach2", 1)) {
        mReceiver.onAttach2();
      }
      return;
    }
    if (event == Lifecycle.Event.ON_DESTROY) {
      if (!hasLogger || logger.approveCall("onDetach", 1)) {
        mReceiver.onDetach();
      }
      return;
    }
  }
}

```
## 其他

前面提到的源码分析均是在 `AppCompatActivity/Fragment` 中进行的，现在说说其他地方的。首先 `support v4` 包中的 `Fragment` 和这个没有多大的区别，仍然是在 `Fragment` 实现了 `LifecycleOwner` 接口，但是没有加入  `ReportFragment` 的操作了，直接在执行生命周期的地方去执行 `mLifecycleRegistry.handleLifecycleEvent`。

那么在普通 Activity 和 Fragment 里怎么做呢？官方文档有一段描述自定义 LifecycleRegistry 的做法可以实现这个：
```Java
public class MyActivity extends Activity implements LifecycleOwner {
    private LifecycleRegistry mLifecycleRegistry;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mLifecycleRegistry = new LifecycleRegistry(this);
        mLifecycleRegistry.markState(Lifecycle.State.CREATED);
    }

    @Override
    public void onStart() {
        super.onStart();
        mLifecycleRegistry.markState(Lifecycle.State.STARTED);
    }

    @NonNull
    @Override
    public Lifecycle getLifecycle() {
        return mLifecycleRegistry;
    }
}
```
