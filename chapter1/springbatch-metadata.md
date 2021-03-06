## 前言

马保国男神, 混元形意太极拳掌门人, 在一次比武中, 年轻人不讲武德, 说打就打, 大意了没有闪而没有使出闪电五连鞭而被大家嘲讽

## Spring Batch 统计性

在定时任务的运行过程中, 需要统计一些基本的数据, 比如本次运行的定时任务的入参, 任务读取, 处理, 跳过, 失败, 写入的次数这些关键数据

## Spring Batch 例子

![](/assets/2020112800.png)

我们先定义一个数据源:

```java
@Configuration
public class DataSourceConfig {

    @Bean("codogDb")
    @ConfigurationProperties(prefix = "spring.datasource.codog")
    @Primary
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}
```

配置我们的 springbatch 使用该数据源:

```java
@Configuration
@EnableBatchProcessing
public class CodogBatchConfig implements BatchConfigurer {

    @Autowired
    @Qualifier("codogDb")
    private DataSource dataSource;

    @Autowired
    @Qualifier("codogJobLauncher")
    private JobLauncher jobLauncher;

    @Autowired
    @Qualifier("codogBatchTransactionManager")
    private PlatformTransactionManager platformTransactionManager;

    @Autowired
    @Qualifier("codogJobRepository")
    private JobRepository jobRepository;

    @Override
    public JobRepository getJobRepository() throws Exception {
        return jobRepository;
    }

    @Override
    public PlatformTransactionManager getTransactionManager() throws Exception {
        return platformTransactionManager;
    }

    @Override
    public JobLauncher getJobLauncher() throws Exception {
        return jobLauncher;
    }

    @Override
    public JobExplorer getJobExplorer() throws Exception {
        return null;
    }

    @Bean("codogJobRepository")
    public JobRepository jobRepository(PlatformTransactionManager batchTransactionManager) throws Exception {
        JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
        factory.setDataSource(dataSource);
        factory.setTransactionManager(batchTransactionManager);
        factory.afterPropertiesSet();
        return factory.getObject();
    }

    @Bean("codogBatchTransactionManager")
    @Primary
    public PlatformTransactionManager batchTransactionManager() {
        return new ResourcelessTransactionManager();
    }

    @Bean("codogJobLauncher")
    public JobLauncher jobLauncher(@Qualifier("codogJobRepository") JobRepository jobRepository) throws Exception {
        SimpleJobLauncher simpleJobLauncher = new SimpleJobLauncher();
        simpleJobLauncher.setJobRepository(jobRepository);
        simpleJobLauncher.afterPropertiesSet();
        return simpleJobLauncher;
    }
}
```

定义一个男孩类用于读:

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Boy {
    private String name;
    private Integer age;
}
```

定义一个男孩的处理器:

```java
@Component
public class BoyProcessor implements ItemProcessor<Boy, String> {

    @Override
    public String process(Boy boy) throws Exception {
        return (boy.getAge() < 5) ? "混元太急拳法伺候" : "年轻人不讲武德, 我大意了没有闪";
    }
}
```

拼装实际效果:

```java
final String JOB_NAME = "demo4BaoguoMa";
List<Boy> boyList = new ArrayList<>();
for (int i = 0; i < 10; i++) {
    boyList.add(new Boy("boy" + i, i));
}
final ListReader<Boy> reader = new ListReader<>(boyList);
final Job girlJob = jobBuilderFactory.get(JOB_NAME)
    .flow(stepBuilderFactory.get(JOB_NAME)
        .<Boy, String>chunk(2).reader(reader)
        .processor(boyProcessor).writer(printWriter).build())
    .end().build();
jobLauncher.run(girlJob, new JobParametersBuilder()
    .addDate("start_time", new Date()).toJobParameters());
```

输出如下:

```
混元太急拳法伺候
混元太急拳法伺候
混元太急拳法伺候
混元太急拳法伺候
混元太急拳法伺候
年轻人不讲武德, 我大意了没有闪
年轻人不讲武德, 我大意了没有闪
年轻人不讲武德, 我大意了没有闪
年轻人不讲武德, 我大意了没有闪
年轻人不讲武德, 我大意了没有闪
```

我们观察两张表:

BATCH\_JOB\_EXECUTION\_PARAMS job参数这表:![](/assets/2020112801.png)BATCH\_STEP\_EXECUTION step执行情况的表:![](/assets/2020112802.png)具体表的逻辑大家可以搭建下情况看看, 还是很好理解的

 

