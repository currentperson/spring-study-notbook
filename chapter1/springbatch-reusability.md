## 前言

最近临近双十一了, 大家都变成尾款人了么, 这次主要解释下 Spring Batch 框架的组件可复用性

## Spring Batch 可复用性

job 分为 step, step 分为 reader, processor, writer, 增加了不同的 job 之间输入/处理/输出能力的复用性

![](/assets/2020101400.png)

所以根据这个模型, 不同的 job 可以使用相同的已经定义好的 step, 不同的 step 和使用相同的已经定义好的 reader, processor 和 writer, 后者因为粒度小, 所以这种情况显然更常见

## Spring Batch 可复用例子

假设我们有一些的商品\(**Commodity**\)付了预付款, 我们需要批量读取这些商品, 将每个商品按照购买的数量, 以一个商品一百元的价格付款, 让我们看看这个过程应该怎么用 Spring Batch 实现吧!

### 定义 Reader\(与之前的文章相同的\)

```java
@Data
public class ListReader<T> implements ItemReader<T> {

    private List<T> sourceList;
    private int index = 0;

    @Override
    public T read() throws Exception {
        return sourceList.size() > index ? sourceList.get(index++) : null;
    }
}
```

### 定义 Processor\(尾款人处理器,这次新写的\)

```java
@Component
public class FinalPaymentProcessor implements ItemProcessor<Commodity, String> {

    @Override
    public String process(Commodity commodity) throws Exception {
        return "商品" + commodity.getName() + "付尾款了" + (commodity.getBuyerCount() * 100);
    }
}
```

### 定义 Writer\(与之前的文章相同的\)

```java
@Component
public class PrintWriter implements ItemWriter<Object> {

    @Override
    public void write(List<?> list) throws Exception {
        list.stream().forEach(System.out::println);
    }
}
```

### 拼装 Job

```java
@Scheduled(cron = "0 0/1 * * * ?")
public void demo4FinalPayments() throws Exception {
    final String JOB_NAME = "demo4FinalPayments";
    List<Commodity> commodityList = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        commodityList.add(new Commodity("商品" + i, i));
    }
    final ListReader<Commodity> reader = new ListReader<>(commodityList);
    final Job girlJob = jobBuilderFactory.get(JOB_NAME)
        .flow(stepBuilderFactory.get(JOB_NAME)
            .<Commodity, String>chunk(2).reader(reader)
            .processor(finalPaymentProcessor).writer(printWriter).build())
        .end().build();
    jobLauncher.run(girlJob, new JobParametersBuilder()
        .addDate("start_time", new Date()).toJobParameters());
}
```

### 输出:

```java
商品商品0付尾款了0
商品商品1付尾款了100
商品商品2付尾款了200
商品商品3付尾款了300
商品商品4付尾款了400
商品商品5付尾款了500
商品商品6付尾款了600
商品商品7付尾款了700
商品商品8付尾款了800
商品商品9付尾款了900
```

## 附录

可以运行的项目代码地址: [https://github.com/currentperson/spring-codog](https://github.com/currentperson/spring-codog)

本系列文章地址: [https://github.com/currentperson/spring-study-notbook ](https://github.com/currentperson/spring-study-notbook )

设计模式系列文章地址: [https://github.com/currentperson/codog-java-design-pattern](https://github.com/currentperson/codog-java-design-pattern)

设计模式项目代码地址: [https://github.com/currentperson/CodogDesignPattern](https://github.com/currentperson/CodogDesignPattern)

