title: Android Studio莫名奇妙错误系列
date: 2015-10-14 15:13:06
tags: ["Android"]
---
## 错误环境

Android studio 在Gradle编译的时候经常会遇到一些莫名其妙的问题。像这个

> ErrorExecution failed for task ‘appdexDebug’. com.android.ide.common.process.ProcessException org.gradle.process.internal.ExecException Process ‘command ‘D:\Java\jdk1.8.0_45binjava.exe’’ finished with non-zero exit value 1

当我把java module添加到app依赖的以后，运行app会报这个错。

## 解决办法

下载Java JDK1.7

打开项目的Project Structure

左侧选择SDK Location

右侧改变JDK location 把1.8改成1.7

## 原因

android studio 不支持jdk1.8
