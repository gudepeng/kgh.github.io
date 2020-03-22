---
layout: post
title: spring boot插件开发实战和原理
categories: sprng
description: sprng factories
keywords: sprng
---
>版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。  
本文链接：[https://gudepeng.github.io/note/2020/03/22/sprngfactories/](https://gudepeng.github.io/note/2020/03/22/sprngfactories/)

## 一.实战：编写spring boot插件
### 1.为什么要编写boot插件
因为我们在开发的时候需要提供一些共同的功能，所以我们编写个共同的jar包。开发人员在使用jar包的时候不用考虑jar包的内容，直接使用具体的功能即可，但是可能由于包路径的不同，你所编写的bean没有被初始化到spring容器中。不应该让开发人员去扫描你的包路径去初始化bean。所以我们要自己动手去把bean初始化到bean容器中，这也是spring扩展能力的由来（spriing.factories）

### 2.实战
编写插件代码,编写配置类（例如：DemoAutoConfig），在其中定义你需要的bean
在resources下创建META-INF/spring.factories
编写spring.factories
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=
com.demo.DemoAutoConfig
```

## 二.spring.factories 常用配置接口
### 1. org.springframework.boot.SpringApplicationRunListener 
SpringApplicationRunListener来监听Spring Boot的启动流程，并且在各个流程中处理自己的逻辑。在应用启动时，在Spring容器初始化的各个阶段回调对应的方法。

### 2. org.springframework.context.ApplicationContextInitializer
ApplicationContextInitializer是在springboot启动过程上下文 ConfigurableApplicationContext刷新方法前(refresh)调用，对ConfigurableApplicationContext的实例做进一步的设置或者处理。

### 3.org.springframework.boot.autoconfigure.EnableAutoConfiguration 
定义系统自动装配的类。

### 4.org.springframework.boot.env.EnvironmentPostProcessor
配置环境的集中管理。比如扩展去做排除加载系统默认的哪些配置类，方便自定义扩展。

## 三.spring factories 原理
### 1.获取配置流程
在启动类注解@SpringBootApplication中可以看到引用了@EnableAutoConfiguration。
其中@Import(AutoConfigurationImportSelector.class)
```
public String[] selectImports(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return NO_IMPORTS;
    }
    AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
            .loadMetadata(this.beanClassLoader);
    AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(autoConfigurationMetadata,
            annotationMetadata);
    return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```
其中getAutoConfigurationEntry方法
```
protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    configurations = removeDuplicates(configurations);
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    configurations = filter(configurations, autoConfigurationMetadata);
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}
```
其中getCandidateConfigurations
```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
            getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
            + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```
调用了SpringFactoriesLoader.loadFactoryNames
```
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
```
loadSpringFactories方法
```
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```
### 2.加载配置流程
在main方法启动的时候我们会调用SpringApplication.run方法
run方法中调用了getSpringFactoriesInstances
调用createSpringFactoriesInstances
```
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,
			ClassLoader classLoader, Object[] args, Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        }
        catch (Throwable ex) {
            throw new IllegalArgumentException("Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```
## 四.总结
这是一种类似插件的设计方式，只要引入对应的jar包，就会扫描到jar里的spring.factories，对应的实现类也就会被实例化。