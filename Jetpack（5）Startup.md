# Jetpack (5) Startup

> [app-startup](https://developer.android.com/topic/libraries/app-startup)은 애플리케이션 시작 시 여러 구성 요소를 초기화하는 간단하고 효율적인 방법을 제공하며, 이를 통해 구성 요소 간의 초기화 순서를 명시적으로 설정하고 애플리케이션 시작 시간을 최적화할 수 있습니다.

```kts
implementation("androidx.startup:startup-runtime:1.1.1")
```

## 필요성

`Startup`은 라이브러리 개발자와 앱 개발자가 동일한 `ContentProvider`를 공유하여 각각의 초기화 로직을 완료할 수 있도록 하며, 컴포넌트 간 초기화 시퀀스 설정을 지원하므로 초기화해야 하는 각 컴포넌트에 대해 별도의 `ContentProvider`를 정의할 필요가 없어 애플리케이션 시작 시간을 크게 단축할 수 있습니다.

많은 외부 라이브러리에서는 개발자의 편의성을 위하여 `context` 객체를 가져오고 초기화 프로세스를 자동으로 완료하기 위해 `ContentProvider`를 선언하도록 선택합니다. 예를 들어, `Lifecycle 컴포넌트`는 `context` 객체를 가져오고 초기화 프로세스를 완료하기 위해 `ProcessLifecycleOwnerInitializer`를 선언합니다. 안드로이드 매니페스트 파일에 선언된 각 `ContentProvider`는 애플리케이션의 `onCreate()`가 호출되기 전에 미리 실행됩니다. 이때 애플리케이션에 `ContentProvider`가 너무 많으면 애플리케이션 시작 시간이 크게 늘어납니다.

`Startup`은 많은 의존성(애플리케이션 자체 구성요소, 외부 구성요소)에 대해 하나로 통합된 초기화 진입점을 제공할 수 있지만, 물론 대부분의 타사 종속 컴포넌트에서 `Startup`이 릴리스되고 채택될 때까지 기다려야 합니다.

## 사용 방법

프로젝트에 세 개의 라이브러리가 있고 초기화해야한다고 가정합니다. 여기서 **Library A**는 **Library B**에 의존하고, **Library B**는 **Library C**에 의존하며, **Library C**는 의존성이 없습니다. 이 경우 세 개의 `Initializer` 구현 클래스를 각각 생성할 수 있습니다.

> A -> B -> C


- Initializer
  - 초기화 로직과 초기화 순서를 선언하기 위해 `Startup`에서 제공하는 인터페이스
  - `create(context: Context)`: 초기화하고 결과를 반환
  - `dependencies()`: `Initializer`를 초기화하기 전에 초기화해야 할 다른 의존성 목록을 반환

```kotlin
class InitializerA : Initializer<A> {

    // 컴포넌트를 초기화하고 초기화 결과 값을 반환합니다.
    override fun create(context: Context): A {
        return A.init(context)
    }

    // 초기화해야 하는 Initializer목록을 가져와서 초기화할 수 있도록 합니다.
    // 다른 컴포넌트에 의존할 필요가 없다면 빈 리스트를 반환할 수 있습니다.
    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(InitializerB::class.java)
    }

}

class InitializerB : Initializer<B> {

    override fun create(context: Context): B {
        return B.init(context)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf(InitializerC::class.java)
    }

}

class InitializerC : Initializer<C> {

    override fun create(context: Context): C {
        return C.init(context)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        return listOf()
    }

}
```

## 초기화 방법

두 가지 초기화 방법이 있으며, **자동 초기화**와 **수동 초기화(지연 초기화)**로 구분됩니다.

### 자동 초기화

`AndroidManifest`파일에서 `Startup`에 제공된 `InitializationProvider`를 선언하고,`meta-data` 태그를 사용하여 `Initializer`구현 클래스의 패키지 이름 경로를 선언합니다. value 값은 반드시 `androidx.startup`이어야 합니다. 이번 예제에서는 `InitializerA`만 선언하면 됩니다, `InitializerB`와 `InitializerC`는 `InitializerA`의 `dependencies()`의 반환 값으로 체인 방식으로 지정할 수 있기 때문입니다.

`android:authorities="${applicationId}.androidx-startup"`에서 `${applicationId}`는 변경하지 않아야 합니다.

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="leavesc.lifecyclecore.core.InitializerA"
        android:value="androidx.startup" />
</provider>
```

위의 단계를 완료하면 애플리케이션을 시작할 때 `Startup`은 지정한 순서대로 초기화를 자동으로 수행합니다. 주의할 점은, `Initializer`간에 의존성이 없고 모든 `Initializer`가 `InitializationProvider`에 의해 자동으로 초기화되길 원한다면, 모든 `Initializer`는 명시적으로 선언되어야 하며, `Initializer`의 초기화 순서는 `provider` 내의 선언 순서와 일치해야 합니다.

### 수동 초기화

대부분의 경우 자동 초기화 방식을 사용하면 되나, 어떠한 경우에는 수동 초기화가 더 나을 수도 있습니다. 예를 들어, **컴포넌트의 초기화 비용(성능 소모 또는 시간 소모)**이 높고 해당 컴포넌트가 **사용되지 않을 수 있는 경우**, 컴포넌트가 사용될 때까지 초기화를 지연시킬 수 있습니다.

수동으로 초기화 할 `Initializer`는 `AndroidManifest`파일에 선언할 필요가 없으며, 다음과 같이 초기화 하면 됩니다.

```kotlin
val result = AppInitializer.getInstance(this).initializeComponent(InitializerA::class.java)
```

`Startup` 내부에서는 `Initializer`의 초기화 결과 값을 **캐싱**하기 때문에, `initializeComponent()`를 여러 번 호출하더라도 최초 한번만 초기화됩니다. 이 메소드는 자동 초기화 시 초기화 결과 값을 가져오는 데도 사용할 수 있습니다.


### 외부 라이브러리의 자동 초기화를 비활성화 하기

> 자동 초기화를 위해 `Startup`을 사용하는 외부 라이브러리의 `Initializer`를 제거하는 방법입니다.

`AndroidManifest`의 병합 규칙을 통해 `Initializer`를 제거할 수 있습니다. (직접 외부 라이브러리의 `AndroidManifest`파일을 수정하는 것은 불가능합니다.)

라이브러리의 `Initializer` 패키지 경로가 `com.example.ExampleLoggerInitializer`이라면, 애플리케이션의 `AndroidManifest`파일에 명시적으로 선언하고 `tools:node="remove"`를 추가하여 `AndroidManifest`파일을 병합할 때 그것을 제거하도록 할 수 있습니다. 이렇게 하면 `Startup`은 지정한 `Initializer`을 자동으로 초기화 하지 않습니다.

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.ExampleLoggerInitializer"
        android:value="androidx.startup"
        tools:node="remove" />
</provider>
```

### 모든 자동 초기화를 비활성화 하기

모든 자동 초기화를 비활성화 하려면, `AndroidManifest`파일에 `tools:node="remove"`를 추가하여 모든 `Startup`을 비활성화 할 수 있습니다.


```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    tools:node="remove" />
```

## Lint

Startup 包含一组 Lint 规则，可用于检查是否已正确定义了组件的初始化程序，可以通过运行 `./gradlew :app:lintDebug` 来执行检查规则

例如，如果项目中声明的 `InitializerB` 没有在 AndroidManifest 中进行声明，且也不包含在其它 Initializer 的依赖项列表里时，通过 Lint 检查就可以看到如下的警告语句：

```xml
Errors found:

xxxx\leavesc\lifecyclecore\core\InitializerHodler.kt:52: Error: Every Initializer needs to be accompanied by a corresponding <meta-data> entry in the AndroidManifest.xml file. [Ensur
eInitializerMetadata]
  class `InitializerB` : Initializer<B> {
  ^
```

## 소스코드 분석

Startup 整个依赖库仅包含五个 Java 文件，整体逻辑比较简单，这里依次介绍下每个文件的作用

## StartupLogger

StartupLogger 是一个日志工具类，用于向控制台输出日志

```java
public final class StartupLogger {

    private StartupLogger() {
        // Does nothing.
    }

    /
     * The log tag.
     */
    private static final String TAG = "StartupLogger";

    /
     * To enable logging set this to true.
     */
    static final boolean DEBUG = false;

    /
     * Info level logging.
     *
     * @param message The message being logged
     */
    public static void i(@NonNull String message) {
        Log.i(TAG, message);
    }

    /
     * Error level logging
     *
     * @param message   The message being logged
     * @param throwable The optional {@link Throwable} exception
     */
    public static void e(@NonNull String message, @Nullable Throwable throwable) {
        Log.e(TAG, message, throwable);
    }
}
```

## StartupException

StartupException 是一个自定义的 RuntimeException 子类，当 Startup 在初始化过程中遇到意外之外的情况时（例如，Initializer 存在循环依赖、Initializer 反射失败等情况），就会抛出 StartupException

```java
public final class StartupException extends RuntimeException {
    public StartupException(@NonNull String message) {
        super(message);
    }

    public StartupException(@NonNull Throwable throwable) {
        super(throwable);
    }

    public StartupException(@NonNull String message, @NonNull Throwable throwable) {
        super(message, throwable);
    }
}
```

## Initializer

Initiaizer 是 Startup 提供的用于声明初始化逻辑和初始化顺序的接口，在 `create(context: Context)`方法中完成初始化过程并返回结果值，在`dependencies()`中指定初始化此 Initializer 前需要先初始化的其它 Initializer 

```java
public interface Initializer<T> {

    /
     * Initializes and a component given the application {@link Context}
     * 
     * @param context The application context.
     */
    @NonNull
    T create(@NonNull Context context);

    /
     * @return A list of dependencies that this {@link Initializer} depends on. This is
     * used to determine initialization order of {@link Initializer}s.
     * <br/>
     * For e.g. if a {@link Initializer} `B` defines another
     * {@link Initializer} `A` as its dependency, then `A` gets initialized before `B`.
     */
    @NonNull
    List<Class<? extends Initializer<?>>> dependencies();
}
```

## InitializationProvider

InitializationProvider 就是需要我们主动声明在 AndroidManifest 文件中的 `ContentProvider`，Startup 的整个初始化逻辑都是在这里进行统一触发的

由于 InitializationProvider 的作用仅是用于统一多个依赖项的初始化入口并获得 Context 对象，所以除了 `onCreate()` 方法会由系统自动调用外，其它方法是没有意义的，如果开发者调用了这几个方法就会直接抛出异常

```java
public final class InitializationProvider extends `ContentProvider` {
    @Override
    public boolean onCreate() {
        Context context = getContext();
        if (context != null) {
            AppInitializer.getInstance(context).discoverAndInitialize();
        } else {
            throw new StartupException("Context cannot be null");
        }
        return true;
    }

    @Nullable
    @Override
    public Cursor query(
            @NonNull Uri uri,
            @Nullable String[] projection,
            @Nullable String selection,
            @Nullable String[] selectionArgs,
            @Nullable String sortOrder) {
        throw new IllegalStateException("Not allowed.");
    }

    @Nullable
    @Override
    public String getType(@NonNull Uri uri) {
        throw new IllegalStateException("Not allowed.");
    }

    @Nullable
    @Override
    public Uri insert(@NonNull Uri uri, @Nullable ContentValues values) {
        throw new IllegalStateException("Not allowed.");
    }

    @Override
    public int delete(
            @NonNull Uri uri,
            @Nullable String selection,
            @Nullable String[] selectionArgs) {
        throw new IllegalStateException("Not allowed.");
    }

    @Override
    public int update(
            @NonNull Uri uri,
            @Nullable ContentValues values,
            @Nullable String selection,
            @Nullable String[] selectionArgs) {
        throw new IllegalStateException("Not allowed.");
    }
}
```

## AppInitializer

AppInitializer 是 Startup 整个库的核心重点，整体代码量不足两百行，AppInitializer 的整体流程是：

- 由 InitializationProvider 传入 Context 对象以此来获得 AppInitializer 唯一实例，并调用 `discoverAndInitialize()` 方法完成所有的自动初始化逻辑
- `discoverAndInitialize()` 方法会先对 InitializationProvider 进行解析，获取到包含的所有 metadata，然后按声明顺序依次反射构建每个 metadata 指向的 Initializer 对象
- 当在初始化某个 Initializer 对象之前，会首先判断其关联的依赖项 dependencies 是否为空。如果为空的话则直接调用其 `create(Context)` 方法进行初始化。如果不为空的话则先对 dependencies 进行初始化，对每个 dependency 均重复此遍历操作，直到不包含  dependencies 的 Initializer 最先初始化完成后才原路返回依次进行初始化，从而保证了 Initializer 之间初始化顺序的有序性
- 当存在这几种情况时，Startup 会抛出异常：Initializer 实现类不包含无参构造方法、Initializer 之间存在循环依赖关系、Initializer 的初始化过程（`create(Context)` 方法）抛出了异常

AppInitializer 对外开放了 `getInstance(@NonNull Context context)` 方法用于获取唯一的静态实例

```java
public final class AppInitializer {

    /
     * 唯一的静态实例
     * The {@link AppInitializer} instance.
     */
    private static AppInitializer sInstance;

    /
     * 同步锁
     * Guards app initialization.
     */
    private static final Object sLock = new Object();

    //用于存储所有已进行初始化了的 Initializer 及对应的初始化结果
    @NonNull
    final Map<Class<?>, Object> mInitialized;

    @NonNull
    final Context mContext;

    /
     * Creates an instance of {@link AppInitializer}
     *
     * @param context The application context
     */
    AppInitializer(@NonNull Context context) {
        mContext = context.getApplicationContext();
        mInitialized = new HashMap<>();
    }

    /
     * @param context The Application {@link Context}
     * @return The instance of {@link AppInitializer} after initialization.
     */
    @NonNull
    @SuppressWarnings("UnusedReturnValue")
    public static AppInitializer getInstance(@NonNull Context context) {
        synchronized (sLock) {
            if (sInstance == null) {
                sInstance = new AppInitializer(context);
            }
            return sInstance;
        }
    }
    
    ···
    
}
```

`discoverAndInitialize()` 方法由 InitializationProvider 进行调用，由其触发所有需要进行默认初始化的依赖项的初始化操作

```java
@SuppressWarnings("unchecked")
void discoverAndInitialize() {
    try {
        Trace.beginSection(SECTION_NAME);

        //获取 InitializationProvider 包含的所有 metadata
        ComponentName provider = new ComponentName(mContext.getPackageName(),
                InitializationProvider.class.getName());
        ProviderInfo providerInfo = mContext.getPackageManager()
                .getProviderInfo(provider, GET_META_DATA);
        Bundle metadata = providerInfo.metaData;

        //获取到字符串 androidx.startup
        //因为 Startup 是以该字符串作为 metaData 的固定 value 来进行遍历的
        //所以如果在 AndroidManifest 文件中声明了不同 value 则不会被初始化
        String startup = mContext.getString(R.string.androidx_startup);

        if (metadata != null) {
            //用于标记正在准备进行初始化的 Initializer
            //用于判断是否存在循环依赖的情况
            Set<Class<?>> initializing = new HashSet<>();
            Set<String> keys = metadata.keySet();
            for (String key : keys) {
                String value = metadata.getString(key, null);
                if (startup.equals(value)) {
                    Class<?> clazz = Class.forName(key);
                    //确保 metaData 声明的包名路径指向的是 Initializer 的实现类
                    if (Initializer.class.isAssignableFrom(clazz)) {
                        Class<? extends Initializer<?>> component =
                                (Class<? extends Initializer<?>>) clazz;
                        if (StartupLogger.DEBUG) {
                            StartupLogger.i(String.format("Discovered %s", key));
                        }
                        //进行实际的初始化过程
                        doInitialize(component, initializing);
                    }
                }
            }
        }
    } catch (PackageManager.NameNotFoundException | ClassNotFoundException exception) {
        throw new StartupException(exception);
    } finally {
        Trace.endSection();
    }
}
```

`doInitialize()` 方法是实际调用了 Initializer 的 `create(context: Context)`的地方，其主要逻辑就是通过嵌套调用的方式来完成所有依赖项的初始化，当判断出存在循环依赖的情况时将抛出异常

```java
@NonNull
@SuppressWarnings({"unchecked", "TypeParameterUnusedInFormals"})
<T> T doInitialize(
        @NonNull Class<? extends Initializer<?>> component,
        @NonNull Set<Class<?>> initializing) {
    synchronized (sLock) {
        boolean isTracingEnabled = Trace.isEnabled();
        try {
            if (isTracingEnabled) {
                // Use the simpleName here because section names would get too big otherwise.
                Trace.beginSection(component.getSimpleName());
            }
            if (initializing.contains(component)) {
                //initializing 包含 component，说明 Initializer 之间存在循环依赖
                //直接抛出异常
                String message = String.format(
                        "Cannot initialize %s. Cycle detected.", component.getName()
                );
                throw new IllegalStateException(message);
            }
            Object result;
            if (!mInitialized.containsKey(component)) {
                //如果 mInitialized 不包含 component
                //说明 component 指向的 Initializer 还未进行初始化
                initializing.add(component);
                try {
                    //通过反射调用 component 的无参构造方法并初始化
                    Object instance = component.getDeclaredConstructor().newInstance();
                    Initializer<?> initializer = (Initializer<?>) instance;
                    //获取 initializer 的依赖项
                    List<Class<? extends Initializer<?>>> dependencies =
                            initializer.dependencies();

                    //如果 initializer 的依赖项 dependencies 不为空
                    //则遍历 dependencies 每个 item 进行初始化
                    if (!dependencies.isEmpty()) {
                        for (Class<? extends Initializer<?>> clazz : dependencies) {
                            if (!mInitialized.containsKey(clazz)) {
                                doInitialize(clazz, initializing);
                            }
                        }
                    }
                    if (StartupLogger.DEBUG) {
                        StartupLogger.i(String.format("Initializing %s", component.getName()));
                    }
                    //进行初始化
                    result = initializer.create(mContext);
                    if (StartupLogger.DEBUG) {
                        StartupLogger.i(String.format("Initialized %s", component.getName()));
                    }
                    //将已经进行初始化的 component 从 initializing 中移除掉
                    //避免误判循环依赖
                    initializing.remove(component);
                    //将初始化结果保存起来
                    mInitialized.put(component, result);
                } catch (Throwable throwable) {
                    throw new StartupException(throwable);
                }
            } else {
                //component 指向的 Initializer 已经进行初始化
                //此处直接获取缓存值直接返回即可
                result = mInitialized.get(component);
            }
            return (T) result;
        } finally {
            Trace.endSection();
        }
    }
}
```

## 주의사항

- `InitializationProvider`의 `onCreate()`는 **메인 스레드**에서 호출되며, 각각의 `Initializer`도 따라서 **메인 스레드**에서 실행됩니다. 이는 초기화 시간이 길고, 별도의 스레드에서 실행해야 하는 컴포넌트에는 적합하지 않습니다. 또한 `Initializer`의 `create(context: Context)`의 본래 목적은 컴포넌트를 초기화하고 초기화 결과 값을 반환하는 것입니다. 만약 이 위치에서 **백그라운드 스레드**로 시간이 많이 소요되는 컴포넌트의 초기화를 실행한다면, 유의미한 결과 값을 반환할 수 없게 되며, 이는 간접적으로 `AppInitializer`를 통해 캐시된 초기화 결과 값을 가져올 수 없게 됩니다.
- 어떤 컴포넌트의 초기화가 다른 시간이 많이 소요되는 컴포넌트(초기화 시간이 길고, 다른 스레드에서 처리)의 결과 값에 의존해야 하는 경우에도, `Startup`은 적합하지 않습니다.