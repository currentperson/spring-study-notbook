## 前言

清华美院一名女学生称男同学通过包的掩护摸自己的 PP 并将男同学的身份信息公开到社交媒体, 导致男同学险些社会性死亡, 后来查了监控才证明了男同学的清白.

## Spring Batch 可校验性

我们经常需要大量的读取数据, 其中有些数据可能不符合我们的预期, 比如从接口读了个用户列表, 年龄字段存在负数, 这种可能就需要中断程序或者跳过处理, 接着处理下一条

## Spring Batch 可校验性例子

reader 和 writer 都是原来的, 我们重新写个 processor:

```java
@Component
public class QingGirlProcessor implements ItemProcessor<Girl, String> {

    @Override
    public String process(Girl girl) throws Exception {
        return girl.getName() + "说被摸了PP";
    }
}
```

使用组合将验证的 processor 和上面的业务的 processor 串联起来:

```java
@Configuration
public class ProcessorConfig {

    @Autowired
    private QingGirlProcessor qingGirlProcessor;

    @Bean
    public BeanValidatingItemProcessor<Girl> girlBeanValidatingItemProcessor() throws Exception {
        BeanValidatingItemProcessor<Girl> validator = new BeanValidatingItemProcessor<>();
        validator.setFilter(true);
        validator.afterPropertiesSet();
        return validator;
    }

    @Bean
    public ItemProcessor<Girl, String> girlStringItemProcessor() throws Exception {
        List<ItemProcessor> list = new ArrayList<>();
        list.add(girlBeanValidatingItemProcessor());
        list.add(qingGirlProcessor);
        CompositeItemProcessor compositeItemProcessor =
            new CompositeItemProcessor<>();
        compositeItemProcessor.setDelegates(list);
        compositeItemProcessor.afterPropertiesSet();

        return compositeItemProcessor;
    }
}
```

任务配置:

```java
final String JOB_NAME = "demo4QingGirl";
List<Girl> girlList = new ArrayList<>();
for (int i = 0; i < 10; i++) {
    girlList.add(i % 2 == 0 ? new Girl() : new Girl("唐靖"));
}
final ListReader<Girl> reader = new ListReader<>(girlList);
final Job girlJob = jobBuilderFactory.get(JOB_NAME)
    .flow(stepBuilderFactory.get(JOB_NAME)
        .<Girl, String>chunk(2).reader(reader)
        .processor(girlStringItemProcessor).writer(printWriter).build())
    .end().build();
jobLauncher.run(girlJob, new JobParametersBuilder()
    .addDate("start_time", new Date()).toJobParameters());
```

输出:

```java
唐靖说被摸了PP
唐靖说被摸了PP
唐靖说被摸了PP
唐靖说被摸了PP
唐靖说被摸了PP
```

总共十个只输出了五个这就是我们的 Girl 类的作用了:

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Girl {

    @NotBlank
    private String name;
}
```

这样就能保证只处理符合预期的数据了

