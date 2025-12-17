---
categories: Android
tag: Retrofit
title: Retrofit 封装实战：多实例缓存与安全请求的 Kotlin 实践
toc: true
toc_sticky: true
---

​	本文介绍一个高效的 Retrofit 封装方案，支持多实例缓存、统一配置、安全请求处理和清晰的错误管理。通过合理的设计决策，保持接口层的纯粹性，同时提供强大的网络请求能力。

[源码地址](https://github.com/che2n3jigw/android-net)

## 核心特性

### 1. 多 Retrofit 实例缓存

使用 `ConcurrentHashMap` 存储不同 BaseUrl 的 Retrofit 实例，避免重复创建，提升性能。

```kotlin
private val mRetrofitMap = ConcurrentHashMap<String, Retrofit>()
```

**优势：**

- 支持多域名场景
- 实例复用，减少内存开销
- 线程安全的并发访问

### 2. 统一 OkHttp 配置

提供标准的网络配置模板，包括超时设置和日志拦截。

```kotlin
private val mLogging by lazy {
    HttpLoggingInterceptor().apply { 
        setLevel(HttpLoggingInterceptor.Level.BODY) 
    }
}

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

**配置项：**

- 连接超时（默认 30s）
- 读取超时（默认 30s）
- 写入超时（默认 30s）
- 可选的日志拦截器
- 自定义拦截器支持

### 3. 智能 JSON 解析

采用 kotlinx.serialization 作为 JSON 解析框架，自动忽略未定义字段，增强兼容性。

```kotlin
private val mJsonFactory by lazy {
    val json = Json { ignoreUnknownKeys = true }
    json.asConverterFactory("application/json; charset=UTF8".toMediaType())
}
```

**特点：**

- 类型安全的序列化
- 自动忽略多余字段，避免解析失败
- 支持 Kotlin 协程原生

## 架构设计

### Retrofit 实例管理

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
): Retrofit
```

**缓存策略：**

- 首次请求创建实例并缓存
- 后续请求直接使用缓存
- 支持手动清除缓存（clearCache = true）

### 简化的 API 服务创建

通过泛型方法，一行代码即可创建接口服务：

```kotlin
inline fun <reified T> createService(
    baseUrl: String,
    connectTimeout: Long = 30_000,
    readTimeout: Long = 30_000,
    writeTimeout: Long = 30_000,
    enableLogging: Boolean = true,
    clearCache: Boolean = false,
    converters: List<Converter.Factory> = emptyList(),
    interceptors: List<Interceptor> = emptyList()
): T
```

**使用示例：**

```kotlin
val apiService = RequestClient.createService<GitHubApi>("https://api.github.com")
```

## 统一的错误处理机制

### 请求结果封装

使用密封类定义清晰的请求结果类型：

```kotlin
@Serializable
sealed class RequestResult<out T> {
    data class Success<T>(val data: T) : RequestResult<T>()
    data class Error(val code: Int, val message: String) : RequestResult<Nothing>()
}
```

**设计优势：**
- 编译器强制检查 when 分支，确保所有状态都被处理
- 类型安全，避免空指针异常
- 清晰的失败信息传递

### 安全的 API 调用

```kotlin
object RequestUtils {
    var globalErrorHandler: (suspend (code: Int, message: String) -> Unit)? = null
    
    suspend fun <T> safeApiCall(apiCall: suspend () -> T?): RequestResult<T> {
        // 统一异常处理和错误包装
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

**异常处理策略：**
1. **HTTP 异常**：获取状态码和错误信息
2. **网络异常**：处理连接超时、无网络等情况
3. **业务异常**：返回 null 数据时的统一处理
4. **全局回调**：支持统一错误上报或提示

## 为何不使用 CallAdapter？

### 保持接口层的纯粹性

我的设计原则是让 API 接口只关注请求定义，不耦合业务封装逻辑：

**传统方式（依赖 CallAdapter）：**

```kotlin
@GET("/users")
suspend fun getUsers(): ApiResponse<List<User>>  // 耦合自定义包装类型
```

**我的方式（保持纯粹）：**

```kotlin
@GET("/repos/{owner}/{repo}/contributors")
suspend fun contributors(
    @Path("owner") owner: String?,
    @Path("repo") repo: String?
): List<Contributor?>?  // 只定义真实数据结构
```

### 调用层封装的优势

```kotlin
val result = RequestUtils.safeApiCall {
    apiService.contributors("square", "retrofit")
}

when (result) {
    is RequestResult.Success -> {
        // 处理成功数据
    }
    is RequestResult.Error -> {
        // 统一错误处理
    }
}
```

**对比优势：**

| 维度     | CallAdapter 方案       | safeApiCall 方案   |
| :------- | :--------------------- | :----------------- |
| 接口定义 | 依赖统一包装类型       | 保持原始数据结构   |
| 异常处理 | 隐蔽在适配器内部       | 显式在调用层处理   |
| 可维护性 | 耦合度高，修改影响面大 | 解耦，修改影响局部 |
| 学习成本 | 需要理解自定义适配器   | 直观的协程调用模式 |
| 灵活性   | 泛型适配器限制多       | 可组合任意数据转换 |

### 设计决策的理由

1. **关注点分离**：接口定义专注于请求规范，错误处理逻辑由通用工具处理
2. **易于调试**：错误处理逻辑显式可见，便于追踪和调试
3. **渐进式改进**：可在不修改接口定义的情况下增强错误处理能力
4. **团队协作**：新人更容易理解和使用，减少学习成本

## 开发中遇到的问题

### retrofit &  okhttp

retrofit 当前版本为3.0.0 ，内部依赖了okhttp 4.12.0 版本

okhttp-interceptor 当前版本为5.2.1 ，内部依赖了okhttp 5.2.1 版本

新版本的okhttp需要在kotlin2.2.0以上编译（OkHttpClient.class）

```kotlin
// This class file was compiled with different version of Kotlin compiler and can't be decompiled.
//
// Current compiler ABI version is 2.0.0
// File ABI version is 2.2.0
```

**解决方案**

1. 降低okhttp-interceptor依赖版本
2. 升级kotlin插件版本，具体参考[Android Studio、Android Gradle 、Android Gradle 插件、kotlin 插件兼容性](https://che2n3jigw.github.io/android/Android-env-compatibility/)