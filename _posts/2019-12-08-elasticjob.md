---
layout: post
title: spring boot集成elastic-job
categories: elastic-job
description: spring boot + elastic-job
keywords: elastic-job, spring boot
---
>版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。 
本文链接：[https://gudepeng.github.io/note/2019/12/08/elasticjob/](https://gudepeng.github.io/note/2019/12/08/elasticjob/)  
demo样例：[https://github.com/gudepeng/demoproject/tree/master/elastic-job](https://github.com/gudepeng/demoproject/tree/master/elastic-job)


### 1.引包
```
<dependency>
    <groupId>com.dangdang</groupId>
    <artifactId>elastic-job-lite-core</artifactId>
    <version>2.1.5</version>
    <exclusions>
        <exclusion>
            <artifactId>*</artifactId>
            <groupId>org.apache.curator</groupId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>2.13.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>2.13.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-x-discovery</artifactId>
    <version>2.13.0</version>
</dependency>
```

### 2.编写配置类
```
@Configuration
@ConfigurationProperties(prefix = "elasticjob")
public class ElasticJobConfig {

    /**
     * 连接Zookeeper服务器的列表. 包括IP地址和端口号. 多个地址用逗号分隔. 如: host1:2181,host2:2181
     */
    private String serverLists;

    /**
     * 命名空间
     */
    private String namespace;


    @Bean(initMethod = "init")
    public ZookeeperRegistryCenter zookeeperRegistryCenter() {
        ZookeeperConfiguration zookeeperConfiguration = new ZookeeperConfiguration(serverLists, namespace);
        return new ZookeeperRegistryCenter(zookeeperConfiguration);
    }

    public String getServerLists() {
        return serverLists;
    }

    public void setServerLists(String serverLists) {
        this.serverLists = serverLists;
    }

    public String getNamespace() {
        return namespace;
    }

    public void setNamespace(String namespace) {
        this.namespace = namespace;
    }

}
```

### 3.编写执行代码
```
public class MyElasticJob implements SimpleJob {
    @Override
    public void execute(ShardingContext shardingContext) {
        System.out.println("AAAAAAAAA");
    }
}
```
集成SimpleJob类，如果是其他类型任务，请集成其他类

### 4.编写注册执行器
```
@Component
public class ElasticJobHandler {
    @Autowired
    private ZookeeperRegistryCenter registryCenter;

    @PostConstruct
    public void addJob(){
        JobCoreConfiguration simpleCoreConfig = JobCoreConfiguration.newBuilder("demoSimpleJob", "0/15 * * * * ?", 10).build();
        // 定义SIMPLE类型配置
        SimpleJobConfiguration simpleJobConfig = new SimpleJobConfiguration(simpleCoreConfig, MyElasticJob.class.getCanonicalName());
        // 定义Lite作业根配置
        LiteJobConfiguration simpleJobRootConfig = LiteJobConfiguration.newBuilder(simpleJobConfig).build();

        new JobScheduler(registryCenter, simpleJobRootConfig).init();
    }
}
```

### 5.创建任务监听器
```
public class ElasticJobListener extends AbstractDistributeOnceElasticJobListener {
    public ElasticJobListener(long startedTimeoutMilliseconds, long completedTimeoutMilliseconds) {
        super(startedTimeoutMilliseconds, completedTimeoutMilliseconds);
    }

    @Override
    public void doBeforeJobExecutedAtLastStarted(ShardingContexts shardingContexts) {
        System.out.println("before");
    }

    @Override
    public void doAfterJobExecutedAtLastCompleted(ShardingContexts shardingContexts) {
        System.out.println("after");
    }
}
```
可以实现ElasticJobListener类，或者集成AbstractDistributeOnceElasticJobListener类。  
ElasticJobListener：每个节点任务执行，都会执行ElasticJobListener实现的方法。  
AbstractDistributeOnceElasticJobListener：只有一个节点执行。  
定义bean  
```
 @Bean
    public MyElasticJobListener myElasticJobListener(){
        return new MyElasticJobListener(0,0);
    }
```
编写完监听器后，需要添加到执行器内。
```
@Autowired
private MyElasticJobListener myElasticJobListener;
...
new JobScheduler(registryCenter, simpleJobRootConfig,myElasticJobListener).init();
...
    
```