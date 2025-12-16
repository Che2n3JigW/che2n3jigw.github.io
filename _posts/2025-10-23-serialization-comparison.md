---
categories: Android
title: Kotlinx-serialization vs Gson vs Moshi：现代JSON序列化框架的抉择
tags:
  - 序列化
  - gson
  - moshi
  - Kotlinx-serialization
toc: true
toc_sticky: true
excerpt: Kotlinx-serialization、Gson和Moshi分别代表了不同设计哲学下的解决方案，本文将深入剖析它们的异同与适用场景。
---

Kotlinx-serialization、Gson和Moshi分别代表了不同设计哲学下的解决方案，本文将深入剖析它们的异同与适用场景。

## 第一章：Gson——Java时代的经典之作

### 1.1 诞生背景与设计理念

Gson由Google在2008年推出，是Java生态系统中最古老、应用最广泛的JSON库之一。其设计哲学是"简单至上"：

```java
// 典型的Gson使用方式
Gson gson = new Gson();
User user = new User("Alice", 30);

// 序列化
String json = gson.toJson(user);
// 输出: {"name":"Alice","age":30}

// 反序列化
User userFromJson = gson.fromJson(json, User.class);
```

### 1.2 核心特性
- **零配置基础使用**：默认情况下无需注解即可工作
- **类型擦除绕过**：通过TypeToken处理泛型

```java
// 处理泛型列表
Type listType = new TypeToken<List<User>>(){}.getType();
List<User> users = gson.fromJson(jsonArray, listType);
```

- **灵活的适配器系统**：可自定义序列化逻辑
- **强大的日期格式化支持**

### 1.3 Kotlin中的局限性

```kotlin
data class User(val name: String, val age: Int)

// Gson可能破坏Kotlin的特性
val user = User(null, 30) // 编译错误：name不可为空

// 但Gson通过反射创建对象时可能绕过空安全检查
val json = """{"age":30}""" // name字段缺失
val user = gson.fromJson(json, User::class.java)
// user.name实际为null，违反了Kotlin非空约束！
```

## 第二章：Moshi——Square公司的现代响应

### 2.1 针对Gson痛点的改进

Moshi由Square公司开发，继承了Gson的简洁性，同时解决了其核心问题：

```kotlin
// Moshi的基本使用
val moshi = Moshi.Builder().build()
val jsonAdapter = moshi.adapter(User::class.java)

val user = User("Alice", 30)
val json = jsonAdapter.toJson(user)

// 明确的空安全处理
val incompleteJson = """{"age":30}"""
val user = jsonAdapter.fromJson(incompleteJson)
// 可能抛出JsonDataException或返回null
```

### 2.2 核心改进点

- **Kotlin友好**：原生支持Kotlin特性
- **更好的空安全**：显式处理空值
- **性能优化**：代码生成选项减少反射使用
- **模块化设计**：更容易扩展

### 2.3 代码生成模式

```kotlin
@JsonClass(generateAdapter = true)
data class User(
    val name: String,
    val age: Int,
    @Json(name = "created_at") val createdAt: Date?
)

// 编译时生成适配器，提高性能
val moshi = Moshi.Builder()
    .add(KotlinJsonAdapterFactory()) // 支持Kotlin特性
    .build()
```

## 第三章：Kotlinx-serialization——原生的Kotlin解决方案

### 3.1 Kotlin-first的设计哲学

Kotlinx-serialization是JetBrains官方提供的序列化框架，完全围绕Kotlin语言特性构建：

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class User(
    val name: String,
    val age: Int,
    @SerialName("created_at") val createdAt: Long? = null
)

fun main() {
    val user = User("Alice", 30, System.currentTimeMillis())
    
    // 序列化
    val json = Json.encodeToString(user)
    // 输出: {"name":"Alice","age":30,"created_at":1633024000000}
    
    // 反序列化
    val decodedUser = Json.decodeFromString<User>(json)
}
```

### 3.2 核心特性亮点

#### 3.2.1 多格式支持

```kotlin
// 不仅仅是JSON
@Serializable
data class Project(val name: String, val stars: Int)

fun main() {
    val project = Project("kotlinx.serialization", 1000)
    
    // JSON
    val json = Json.encodeToString(project)
    
    // CBOR (简洁的二进制格式)
    val cbor = Cbor.encodeToByteArray(project)
    
    // Protobuf
    val protoBuf = ProtoBuf.encodeToByteArray(project)
    
    // 属性列表
    val properties = Properties.encodeToStringMap(project)
}
```

#### 3.2.2 编译时安全

```kotlin
// 编译时类型检查
@Serializable
data class User(
    val name: String,
    val age: Int,
    val email: String? = null // 可选字段
)

// 缺失必填字段会在编译时或运行时明确报错
val invalidJson = """{"name":"Alice"}"""
// Json.decodeFromString<User>(invalidJson) 
// 抛出: SerializationException: Field 'age' is required
```

#### 3.2.3 密封类和多态序列化

```kotlin
@Serializable
sealed class Response

@Serializable
@SerialName("success")
data class Success(val data: String) : Response()

@Serializable
@SerialName("error")
data class Error(val message: String) : Response()

fun main() {
    val responses = listOf<Response>(
        Success("Data loaded"),
        Error("Network failure")
    )
    
    val json = Json.encodeToString(responses)
    // 输出: [{"type":"success","data":"Data loaded"},{"type":"error","message":"Network failure"}]
}
```

## 第四章：三大框架全方位对比

### 4.1 技术架构对比

| 特性维度       | Gson               | Moshi                  | Kotlinx-serialization   |
| :------------- | :----------------- | :--------------------- | :---------------------- |
| **设计理念**   | Java优先，反射驱动 | Gson改进版，Kotlin友好 | Kotlin原生，多平台      |
| **序列化方式** | 运行时反射         | 反射或代码生成         | 编译时生成              |
| **Kotlin支持** | 需要适配器         | 良好，需要工厂         | 完美，语言级集成        |
| **空安全**     | 差                 | 良好                   | 优秀                    |
| **默认值支持** | 有限               | 良好                   | 优秀                    |
| **多平台**     | JVM为主            | JVM为主                | 全平台（JVM/JS/Native） |
| **依赖大小**   | ~240KB             | ~270KB                 | ~800KB（含多格式）      |

### 4.2 性能基准对比（相对值）

**序列化速度**：Kotlinx-serialization (代码生成) > Moshi (代码生成) > Moshi (反射) > Gson
{: .notice--info}

**反序列化速度**：Moshi (代码生成) ≈ Kotlinx-serialization > Moshi (反射) > Gson
{: .notice--info}

**内存使用**：Kotlinx-serialization < Moshi < Gson
{: .notice--info}

**启动性能**：Kotlinx-serialization > Moshi (代码生成) > Gson> Moshi (反射) 
{: .notice--info}

## 第五章：实战场景选择指南

### 5.1 新项目技术选型决策树

<div class="notice">
  <h4>项目类型 → Kotlin使用程度 → 平台需求 → 推荐方案</h4>
  	<p>├── 纯Kotlin新项目 → 全平台 → kotlinx-serialization</p>
	<p>├── 纯Kotlin新项目 → Android/JVM → Moshi或kotlinx-serialization</p>
	<p>├── Java项目 → 少量Kotlin → Gson或Moshi</p>
	<p>└── 混合/遗留项目 → 逐步迁移 → 根据模块选择</p>
</div>

## 第六章：高级特性深度解析

### 6.1 kotlinx-serialization的上下文序列化

```kotlin
// 处理无法添加@Serializable注解的类
object DateSerializer : KSerializer<Date> {
    override val descriptor = PrimitiveSerialDescriptor("Date", PrimitiveKind.STRING)
    
    override fun serialize(encoder: Encoder, value: Date) {
        encoder.encodeString(value.toString())
    }
    
    override fun deserialize(decoder: Decoder): Date {
        return SimpleDateFormat("yyyy-MM-dd").parse(decoder.decodeString())
    }
}

@Serializable
data class Event(val name: String, val date: @Contextual Date)

fun main() {
    val json = Json { serializersModule = SerializersModule {
        contextual(Date::class, DateSerializer)
    }}
    
	json.encodeToString(Event("name", Date()))
    // {"name":"name","date":"Tue Dec 16 14:07:15 GMT+08:00 2025"}
    
    val string = """
    	{"name":"name","date":"2025-12-16"}
    """.trimIndent()
    json.decodeFromString<Event>(string)
    // Event(name=name, date=Tue Dec 16 00:00:00 GMT+08:00 2025)
}
```

### 6.2 Moshi的Kotlin代码生成器
```kotlin
// 使用注解处理器提升性能
@JsonClass(generateAdapter = true)
data class User(val name: String, val age: Int)

// 生成的代码（简化版）
class UserJsonAdapter(moshi: Moshi) : JsonAdapter<User>() {
    private val stringAdapter = moshi.nextAdapter<String>(...)
    private val intAdapter = moshi.nextAdapter<Int>(...)
    
    override fun fromJson(reader: JsonReader): User {
        var name: String? = null
        var age: Int? = null
        reader.beginObject()
        while (reader.hasNext()) {
            when (reader.nextName()) {
                "name" -> name = stringAdapter.fromJson(reader)
                "age" -> age = intAdapter.fromJson(reader)
            }
        }
        reader.endObject()
        return User(name!!, age!!)
    }
}
```

### 6.3 Gson的TypeAdapterFactory高级用法
```java
// 自定义复杂类型的处理
public class AnimalAdapterFactory implements TypeAdapterFactory {
    public <T> TypeAdapter<T> create(Gson gson, TypeToken<T> type) {
        if (!Animal.class.isAssignableFrom(type.getRawType())) {
            return null;
        }
        
        return (TypeAdapter<T>) new TypeAdapter<Animal>() {
            public void write(JsonWriter out, Animal value) {
                // 自定义序列化逻辑
            }
            
            public Animal read(JsonReader in) {
                // 自定义反序列化逻辑
            }
        };
    }
}
```

## 第七章：选择建议
### 7.1 Kotlinx-serialization
- 纯Kotlin项目
- 需要多格式序列化
- 追求最佳性能和类型安全
- 使用Kotlin协程或Compose

### 7.2 Moshi
- 需要与Java代码互操作
- 团队熟悉Square生态
- 需要高度可定制的序列化逻辑

### 7.3 Gson 
- 旧项目维护
- 快速原型验证
- 对性能要求不高
- 团队Java背景为主

**总结:**无论选择哪个框架，关键是根据项目需求、团队技能和长期维护计划做出理性决策。技术选型没有绝对的对错，只有适合与否。在序列化这个看似简单但实则复杂的问题上，正确的工具选择可以显著提升开发效率和系统稳定性。
{: .notice--success}

