---
layout: post
title: spring boot插件开发实战和原理(二)：排除不想使用的自动装配类
categories: sprng
description: sprng factories
keywords: sprng
---
>版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。  
本文链接：[https://gudepeng.github.io/note/2020/04/12/sprngfactories2/](https://gudepeng.github.io/note/2020/04/12/sprngfactories2)

## 一.场景
当引入了一个spring stater的时候不像启用spring.factories中特定的EnableAutoConfiguration类时。
或者咱们开发的stater依赖于其他stater，但是不想启动他的自动装配类时。

## 二.实战
### 1.在自定义的stater内创建TestConfig.java类,实现AutoConfigurationImportFilter接口。
```
public class TestConfig implements AutoConfigurationImportFilter {

    @Override
    public boolean[] match(String[] autoConfigurationClasses, AutoConfigurationMetadata autoConfigurationMetadata) {
        return new boolean[0];
    }
}
```
autoConfigurationClasses数组内是所有自动装配类的全路径。
如果想排除对应的类不使用，只需要在return出去的数组对应位置设置成false。
### 2.在spring.factories添加配置。
```
org.springframework.boot.autoconfigure.AutoConfigurationImportFilter=\
xxx.xxx.TestConfig
```
xxx.xxx.TestConfig 为TestConfig的全路径