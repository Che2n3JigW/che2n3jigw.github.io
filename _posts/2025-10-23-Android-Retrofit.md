---
categories: Android
title: Android Retrofit 二次封装
toc: true
toc_sticky: true
---

# 使用示例

先看看封装完成后的实际使用效果，看看是否满足你的心理预期。

## 接口定义

```kotlin
interface DemoService {
    @GET("/repos/{owner}/{repo}/contributors")
    suspend fun contributors(
        @Path("owner") owner: String?,
        @Path("repo") repo: String?
    ): List<Contributor?>?
}
```

## 请求接口

```kotlin
class DemoRepository {
    private val service = RequestClient.createService<DemoService>(Constants.BASE_URL)

    suspend fun getContributors(): List<Contributor?> {
        val result = RequestUtils.safeApiCall {
            service.contributors("square", "retrofit")
        }
        if (result is RequestResult.Success) {
            return result.data
        }
        return emptyList()
    }
}
```

## 全局异常监听(可选)

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        RequestUtils.globalErrorHandler = { code, message ->
            Toast.makeText(this, "code: $code, message: $message", Toast.LENGTH_SHORT).show()
        }
    }
}
```

## 扩展用法

```kotlin
private val service1 = RequestClient.createService<DemoService>(
    baseUrl = "https://server1.com/",
    enableLogging = false,
    clearCache = true,
    // 自定义实体类转换器
    converters = listOf(CustomConverter()),
    // 自定义拦截器
    interceptors = listOf(CustomInterceptor())
)

private val service2 = RequestClient.createService<DemoService>(
    baseUrl = "https://server2.com/",
    enableLogging = false,
    clearCache = true,
    // 自定义实体类转换器
    converters = listOf(CustomConverter()),
    // 自定义拦截器
    interceptors = listOf(CustomInterceptor())
)

...
```


# ---下面开始封装---



# 添加依赖

```toml
[versions]
# Android Gradle 插件版本
agp = "8.13.0"
# kotlin 插件版本
kotlin = "2.2.20"
# kotlin 序列化依赖和插件版本
kotlinxSerializationJson = "1.9.0"
# kotlin协程
kotlinxCoroutinesCore = "1.10.2"
# okhttp 拦截器依赖版本
okhttp = "5.2.1"
# retrofit 依赖版本
retrofit = "3.0.0"

[libraries]
converter-kotlinx-serialization = { module = "com.squareup.retrofit2:converter-kotlinx-serialization", version.ref = "retrofit" }
kotlinx-coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "kotlinxCoroutinesCore" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinxSerializationJson" }
logging-interceptor = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "okhttp" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlinx-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

## 兼容性问题: retrofit &  okhttp

retrofit 当前版本为3.0.0 ，内部依赖了okhttp 4.12.0 版本

okhttp-interceptor 当前版本为5.2.1 ，内部依赖了okhttp 5.2.1 版本

新版本的okhttp需要在kotlin2.2.0以上编译（OkHttpClient.class）

```kotlin
// This class file was compiled with different version of Kotlin compiler and can't be decompiled.
//
// Current compiler ABI version is 2.0.0
// File ABI version is 2.2.0
```

### 解决方案

1. 降低okhttp-interceptor依赖版本
2. 升级kotlin插件版本，具体参考[Android Studio、Android Gradle 、Android Gradle 插件、kotlin 插件兼容性 - 博客](https://che2n3jigw.github.io/android/Android-env-compatibility/)



# 封装RequestClient

>1. **多 Retrofit 实例缓存**：使用 `ConcurrentHashMap` 存储不同 BaseUrl 的 Retrofit 实例，避免重复创建。
>2. **统一 OkHttp 配置**：提供日志拦截器、连接/读取/写入超时等统一配置。
>3. **JSON 自动解析**：内置 kotlinx.serialization 的 JSON Converter，自动忽略未定义字段。
>4. **可扩展性**：支持自定义拦截器和转换器，满足复杂场景。
>5. **泛型 API 创建**：通过 `createService<T>()` 泛型方法，一行代码即可创建接口服务。

## Retrofit 缓存

```kotlin
private val mRetrofitMap = ConcurrentHashMap<String, Retrofit>()
```

- 用于存储不同 `baseUrl` 的 Retrofit 实例，保证复用和性能。

## 日志拦截器

```kotlin
private val mLogging by lazy {
    HttpLoggingInterceptor().apply { setLevel(HttpLoggingInterceptor.Level.BODY) }
}
```

- 用于打印请求和响应，方便调试。

## JSON 转换器

```kotlin
private val mJsonFactory by lazy {
    val json = Json { ignoreUnknownKeys = true }
    json.asConverterFactory("application/json; charset=UTF8".toMediaType())
}
```

- 使用 kotlinx.serialization 处理 JSON，并忽略多余字段。

## 创建 Retrofit

```kotlin
private fun createRetrofit(
    baseUrl: String,
    connectTimeout: Long = 30_000,
    readTimeout: Long = 30_000,
    writeTimeout: Long = 30_000,
    enableLogging: Boolean = true,
    converters: List<Converter.Factory> = emptyList(),
    interceptors: List<Interceptor> = emptyList()
): Retrofit {
    val builder = Retrofit.Builder().apply {
        // 域名
        baseUrl(baseUrl)
        // OkHttp客户端
        val okHttpClient = provideOkHttpClient(
            connectTimeout, readTimeout, writeTimeout, enableLogging, interceptors
        )
        client(okHttpClient)
        // JSON转换器
        if (converters.isNotEmpty()) {
            for (factory in converters) {
                addConverterFactory(factory)
            }
        }
        addConverterFactory(mJsonFactory)
    }
    return builder.build()
}
```

- 统一创建 Retrofit 实例
- 可自定义超时、拦截器、转换器
- 自动添加 JSON 转换器和 OkHttp 客户端



## 创建 OkHttpClient

```kotlin
private fun provideOkHttpClient(
    connectTimeout: Long,
    readTimeout: Long,
    writeTimeout: Long,
    enableLogging: Boolean = true,
    interceptors: List<Interceptor> = emptyList()
): OkHttpClient {
    val builder = OkHttpClient.Builder().apply {
        connectTimeout(connectTimeout, TimeUnit.MILLISECONDS)
        readTimeout(readTimeout, TimeUnit.MILLISECONDS)
        writeTimeout(writeTimeout, TimeUnit.MILLISECONDS)
        // 添加日志拦截器
        if (enableLogging) {
            addInterceptor(mLogging)
        }
        if (interceptors.isNotEmpty()) {
            for (interceptor in interceptors) {
                addInterceptor(interceptor)
            }
        }
    }
    return builder.build()
}
```

- 设置连接、读取、写入超时
- 可添加日志和自定义拦截器



## 获取 Retrofit & 创建 API 服务

```kotlin
fun getRetrofit(
    baseUrl: String,
    connectTimeout: Long = 30_000,
    readTimeout: Long = 30_000,
    writeTimeout: Long = 30_000,
    enableLogging: Boolean = true,
    clearCache: Boolean = false,
    converters: List<Converter.Factory> = emptyList(),
    interceptors: List<Interceptor> = emptyList()
): Retrofit {
    if (clearCache) {
        mRetrofitMap.remove(baseUrl)
    }
    return mRetrofitMap.getOrPut(baseUrl) {
        createRetrofit(
            baseUrl,
            connectTimeout,
            readTimeout,
            writeTimeout,
            enableLogging,
            converters,
            interceptors
        )
    }
}

inline fun <reified T> createService(
    baseUrl: String,
    connectTimeout: Long = 30_000,
    readTimeout: Long = 30_000,
    writeTimeout: Long = 30_000,
    enableLogging: Boolean = true,
    clearCache: Boolean = false,
    converters: List<Converter.Factory> = emptyList(),
    interceptors: List<Interceptor> = emptyList()
): T {
    return getRetrofit(
        baseUrl,
        connectTimeout,
        readTimeout,
        writeTimeout,
        enableLogging,
        clearCache,
        converters,
        interceptors
    ).create(T::class.java)
}
```

- `getRetrofit()`：获取或缓存 Retrofit 实例
- `createService<T>()`：直接创建接口服务，调用简单



# 处理全局异常

## 定义请求结果类型

我们使用 **sealed class** 定义一个通用的请求结果 `RequestResult`，包含成功和失败两种情况：

```kotlin
@Serializable
sealed class RequestResult<out T> {

    data class Success<T>(val data: T) : RequestResult<T>()

    /**
     * @param code      httpCode 或 自定义错误码
     * @param message   错误信息
     */
    data class Error(val code: Int, val message: String) : RequestResult<Nothing>()
}
```

**说明：**

- `Success<T>`：封装成功返回的数据。
- `Error`：封装失败信息，包括 HTTP 状态码或自定义错误码，以及错误信息。
- 使用 `sealed class` 的好处是，编译器可以强制检查 `when` 分支，使得调用者处理结果时更安全。

## 安全的 API 调用工具 `RequestUtils`

为了减少重复的 try/catch 和统一错误处理，我们封装了一个工具类：

```kotlin
object RequestUtils {

    /**
     * 全局异常回调接口
     */
    var globalErrorHandler: (suspend (code: Int, message: String) -> Unit)? = null

    /**
     * 安全的 API 调用
     * @return RequestResult<T> 成功返回 Success，失败返回 Error
     */
    suspend fun <T> safeApiCall(apiCall: suspend () -> T?): RequestResult<T> {
        return try {
            val result = withContext(Dispatchers.IO) {
                apiCall()
            }
            if (result != null) {
                RequestResult.Success(result)
            } else {
                RequestResult.Error(-1, "No data")
            }
        } catch (e: HttpException) {
            val code = e.code()
            val message = e.message() ?: "Http error"
            globalErrorHandler?.invoke(code, message)
            RequestResult.Error(code, message)
        } catch (e: IOException) {
            val code = -1
            val message = "Network error: ${e.message}"
            globalErrorHandler?.invoke(code, message)
            RequestResult.Error(code, message)
        } catch (e: Exception) {
            val code = -1
            val message = e.message ?: "Unknown error"
            globalErrorHandler?.invoke(code, message)
            RequestResult.Error(code, message)
        }
    }
}
```

**关键点解释：**

1. **统一错误封装**
   - HTTP 错误 → `HttpException`
   - 网络错误 → `IOException`
   - 其他异常 → `Exception`
      所有异常都会返回 `RequestResult.Error`。
2. **全局错误回调**
   - `globalErrorHandler` 可以在 App 启动时统一设置，例如显示全局 Toast 或日志上报。
3. **空数据处理**
   - 当接口返回 `null` 时，返回 `RequestResult.Error(-1, "No data")`，避免调用方拿到空数据而崩溃。

## 为什么没有使用 CallAdapter？

在很多 Retrofit 封装中，我们会看到有人定义一个 `CallAdapter`，让接口可以直接返回类似：

```kotlin
@GET("/users")
fun getUsers(): Call<ApiResponse<List<User>>>
```

或者：

```kotlin
@GET("/users")
suspend fun getUsers(): ApiResponse<List<User>>
```

看起来很优雅，但实际上，这样做需要开发者在定义接口时**必须知道项目自定义的响应包装类型**（比如 `ApiResponse` 或 `BaseResult`）。
 这会让接口定义和业务封装**产生耦合**。

------

### 我选择不使用 CallAdapter 的原因

我希望接口层（也就是 `ApiService`）保持最原始、最干净的声明方式：

```kotlin
@GET("/repos/{owner}/{repo}/contributors")
suspend fun contributors(
    @Path("owner") owner: String?,
    @Path("repo") repo: String?
): List<Contributor?>?
```

这样接口的职责就很单纯：

- 它只定义“请求路径 + 参数 + 返回数据结构”；
- 不关心异常包装、请求状态、错误码处理；
- 不依赖项目内部的响应封装逻辑（比如 `RequestResult`）。

------

### 异常与状态的统一交给外层封装处理

在我们的封装方案中，**接口层只负责描述网络请求**，而请求的“安全调用”与“结果包装”交给 `RequestUtils.safeApiCall` 来完成：

```kotlin
val result = RequestUtils.safeApiCall {
    apiService.contributors("square", "retrofit")
}
```

`safeApiCall` 会自动：

- 捕获 `HttpException`、`IOException`、`Exception` 等异常；
- 返回统一的 `RequestResult.Success` 或 `RequestResult.Error`；
- 触发全局错误回调。

------

### 这样做的好处

| 对比项   | 使用 CallAdapter                       | 使用 safeApiCall 封装         |
| -------- | -------------------------------------- | ----------------------------- |
| 接口定义 | 必须依赖统一包装类型（如 ApiResponse） | 保持干净，只定义真实返回类型  |
| 异常处理 | 在自定义 CallAdapter 内部处理          | 在调用层统一捕获              |
| 可读性   | 封装更隐蔽，初学者难追踪               | 清晰显式，逻辑一目了然        |
| 灵活性   | 需要自定义泛型适配器                   | 可直接组合协程 + 任意数据类型 |

### 总结一句话：

> 不使用 CallAdapter，是为了让 **接口定义尽可能纯粹**，
>  让 **异常包装逻辑下沉到通用层（RequestUtils）**，
>  从而让接口与业务逻辑解耦，调用体验更简单、直观。





# 源码解析

## Retrofit 是如何识别 `suspend` 函数的？

### Retrofit.class

```java
public <T> T create(final Class<T> service) {
	validateServiceInterface(service);
	...
}
private void validateServiceInterface(Class<?> service) {
    ...
	loadServiceMethod(service, method);
    ...
}
ServiceMethod<?> loadServiceMethod(Class<?> service, Method method) {
    ...
	result = ServiceMethod.parseAnnotations(this, service, method);
    ...
}
```

### ServiceMethod.class

```java
static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Class<?> service, Method method) {
    ...
	RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, service, method);
    ...
}
```

### （关键）RequestFactory.class

```java
static RequestFactory parseAnnotations(Retrofit retrofit, Class<?> service, Method method) {
	return new Builder(retrofit, service, method).build();
}

RequestFactory build() {
    ...
    parameterHandlers[p] =
        parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
    ...
}

private @Nullable ParameterHandler<?> parseParameter(
    int p, Type parameterType, 
    @Nullable Annotation[] annotations, 
    boolean allowContinuation) {
    ...
	if (Utils.getRawType(parameterType) == Continuation.class) {
        isKotlinSuspendFunction = true;
        return null;
    }
    ...
}
```

在 Kotlin 编译后的字节码中，每一个 `suspend` 函数的签名都会多一个隐藏参数：

```java
Continuation<? super T> continuation
```

Retrofit 就是通过检测方法参数列表中是否包含 `Continuation` 类型来判断该函数是否为挂起函数的。





## 请求流程

### Retrofit.class

```java
public <T> T create(final Class<T> service) {
    return (T) Proxy.newProxyInstance(
        service.getClassLoader(),
        new Class<?>[] {service},
        new InvocationHandler() {
        	private final Object[] emptyArgs = new Object[0];

            @Override
            public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                Reflection reflection = Platform.reflection;
                return reflection.isDefaultMethod(method)
                    ? reflection.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(service, method).invoke(proxy, args);
            }
    });
}
```

生成一个实现了 service 接口的“代理类”。当你调用任何方法时（比如 api.getUsers()）,实际上都会转发到 InvocationHandler.invoke()。

### HttpServiceMethod.class

```java
  @Override
  final @Nullable ReturnT invoke(Object instance, Object[] args) {
    Call<ResponseT> call =
        new OkHttpCall<>(requestFactory, instance, args, callFactory, responseConverter);
    return adapt(call, args);
  }

```

### SuspendForBody.class

```java
    @Override
    protected Object adapt(Call<ResponseT> call, Object[] args) {
      call = callAdapter.adapt(call);

      //noinspection unchecked Checked by reflection inside RequestFactory.
      Continuation<ResponseT> continuation = (Continuation<ResponseT>) args[args.length - 1];

      try {
        if (isUnit) {
          //noinspection unchecked Checked by isUnit
          return KotlinExtensions.awaitUnit((Call<Unit>) call, (Continuation<Unit>) continuation);
        } else if (isNullable) {
          return KotlinExtensions.awaitNullable(call, continuation);
        } else {
          return KotlinExtensions.await(call, continuation);
        }
      } catch (VirtualMachineError | ThreadDeath | LinkageError e) {
        // Do not attempt to capture fatal throwables. This list is derived RxJava's `throwIfFatal`.
        throw e;
      } catch (Throwable e) {
        return KotlinExtensions.suspendAndThrow(e, continuation);
      }
    }
```

### (核心)KotlinExtensions.kt

```kotlin
suspend fun <T : Any> Call<T>.await(): T {
  return suspendCancellableCoroutine { continuation ->
    continuation.invokeOnCancellation {
      cancel()
    }
    enqueue(object : Callback<T> {
      override fun onResponse(call: Call<T>, response: Response<T>) {
        if (response.isSuccessful) {
          val body = response.body()
          if (body == null) {
            val invocation = call.request().tag(Invocation::class.java)!!
            val service = invocation.service()
            val method = invocation.method()
            val e = KotlinNullPointerException(
              "Response from ${service.name}.${method.name}" +
                " was null but response body type was declared as non-null",
            )
            continuation.resumeWithException(e)
          } else {
            continuation.resume(body)
          }
        } else {
          continuation.resumeWithException(HttpException(response))
        }
      }

      override fun onFailure(call: Call<T>, t: Throwable) {
        continuation.resumeWithException(t)
      }
    })
  }
}
```

这段代码是 Retrofit “将回调转成挂起函数”的关键实现：

| 作用         | 实现                                  |
| ------------ | ------------------------------------- |
| 启动网络请求 | `enqueue()` 异步执行                  |
| 成功回调     | `continuation.resume(body)`           |
| 失败回调     | `continuation.resumeWithException(t)` |
| 支持取消     | `invokeOnCancellation { cancel() }`   |
| 转换为挂起   | 使用 `suspendCancellableCoroutine`    |

这样一来，Retrofit 就能在 Kotlin 世界里**完美支持挂起函数**调用。