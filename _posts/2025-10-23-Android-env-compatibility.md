---
categories: Android
title: Android Studio、Android Gradle 、Android Gradle 插件、kotlin 插件兼容性
toc: true
toc_sticky: true
---

## Android Studio 与 Android Gradle 插件的兼容性

| Android Studio 版本            | 所需的 AGP 版本 |
| :----------------------------- | :-------------- |
| Narwhal 4 功能更新 \| 2025.1.4 | 4.0-8.13        |
| Narwhal 3 功能更新 \| 2025.1.3 | 4.0-8.13        |
| Narwhal 功能更新 \| 2025.1.2   | 4.0-8.12        |
| Narwhal \| 2025.1.1            | 3.2-8.11        |
| 猫鼬功能更新 \| 2024.3.2       | 3.2-8.10        |
| Meerkat \| 2024.3.1            | 3.2-8.9         |
| Ladybug 功能更新 \| 2024.2.2   | 3.2-8.8         |
| Ladybug \| 2024.2.1            | 3.2-8.7         |
| Koala 功能更新 \| 2024.1.2     | 3.2-8.6         |
| Koala \| 2024.1.1              | 3.2-8.5         |
| Jellyfish \| 2023.3.1          | 3.2-8.4         |
| Iguana \| 2023.2.1             | 3.2-8.3         |
| Hedgehog \| 2023.1.1           | 3.2-8.2         |
| Giraffe \| 2022.3.1            | 3.2-8.1         |
| Flamingo \| 2022.2.1           | 3.2-8.0         |

数据来源：[android_gradle_plugin_and_android_studio_compatibility](https://developer.android.com/build/releases/gradle-plugin?hl=zh-cn#android_gradle_plugin_and_android_studio_compatibility)

## Android Gradle 与 Android Gradle 插件兼容性

| 插件版本 | 所需的最低 Gradle 版本 |
| :------- | :--------------------- |
| 8.13     | 8.13                   |
| 8.12     | 8.13                   |
| 8.11     | 8.13                   |
| 8.10     | 8.11.1                 |
| 8.9      | 8.11.1                 |
| 8.8      | 8.10.2                 |
| 8.7      | 8.9                    |
| 8.6      | 8.7                    |
| 8.5      | 8.7                    |
| 8.4      | 8.6                    |
| 8.3      | 8.4                    |
| 8.2      | 8.2                    |
| 8.1      | 8.0                    |
| 8.0      | 8.0                    |

数据来源：[updating-gradle](https://developer.android.com/build/releases/gradle-plugin?hl=zh-cn#updating-gradle)

## Android Gradle与kotlin 插件兼容性

| Kotlin 插件版本 | 最低 Gradle 版本 | Kotlin 语言版本 |
| :-------------- | :--------------- | :-------------- |
| 1.3.10          | 5.0              | 1.3             |
| 1.3.11          | 5.1              | 1.3             |
| 1.3.20          | 5.2              | 1.3             |
| 1.3.21          | 5.3              | 1.3             |
| 1.3.31          | 5.5              | 1.3             |
| 1.3.41          | 5.6              | 1.3             |
| 1.3.50          | 6.0              | 1.3             |
| 1.3.61          | 6.1              | 1.3             |
| 1.3.70          | 6.3              | 1.3             |
| 1.3.71          | 6.4              | 1.3             |
| 1.3.72          | 6.5              | 1.3             |
| 1.4.20          | 6.8              | 1.3             |
| 1.4.31          | 7.0              | 1.4             |
| 1.5.21          | 7.2              | 1.4             |
| 1.5.31          | 7.3              | 1.4             |
| 1.6.21          | 7.5              | 1.4             |
| 1.7.10          | 7.6              | 1.4             |
| 1.8.10          | 8.0              | 1.8             |
| 1.8.20          | 8.2              | 1.8             |
| 1.9.0           | 8.3              | 1.8             |
| 1.9.10          | 8.4              | 1.8             |
| 1.9.20          | 8.5              | 1.8             |
| 1.9.22          | 8.7              | 1.8             |
| 1.9.23          | 8.9              | 1.8             |
| 1.9.24          | 8.10             | 1.8             |
| 2.0.20          | 8.11             | 1.8             |
| 2.0.21          | 8.12             | 1.8             |
| 2.2.0           | 9.0.0            | 2.2             |

数据来源: [Compatibility Matrix](https://docs.gradle.org/current/userguide/compatibility.html#kotlin)

## 查看版本

### Android Studio
{% 
include figure popup=true image_path="/assets/images/android/compatibility/check_android_studio_version1.png" 
%}

{% 
include figure popup=true image_path="/assets/images/android/compatibility/check_android_studio_version2.png" 
%}

### Android Gradle

./gradle/wrapper/gradle-wrapper.properties

{% 
include figure popup=true image_path="/assets/images/android/compatibility/check_gradle_version.png" 
%}

> 版本号为8.13

### Android Gradle 插件(AGP)

./gradle/libs.versions.toml

{% 
include figure popup=true image_path="/assets/images/android/compatibility/check_gradle_plugins_version.png" 
%}

> 版本号为8.13

### kotlin 插件

./gradle/libs.versions.toml

{% 
include figure popup=true image_path="/assets/images/android/compatibility/check_kotlin_plugins_version.png" 
%}

> 版本号为2.2.20