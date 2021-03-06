## 前言

这段时间上海名媛群成为了大家茶余饭后的谈资, 什么花了 500 元卧底上海名媛群, 本来以为能见识到纸醉金迷的上海人上人生活, 结果换来了一堆拼高端酒店, 拼下午茶, 甚至拼二手丝袜的...... 但是仍然瞧不起开宝马的一堆名媛, 今天用这个热门事件作为 Spring Batch 的入门例子

## Spring Batch 介绍

Spring Batch 是一种批处理框架, 不是定时任务触发框架, 比如使用 @Scheduled, Quartz, ElasticJob 等框架都能定时触发任务, 但是任务的内容编写除了写普通的增删改查的 service, 还可以使用 Spring Batch 来编写, 主要优势如下:

1. **模型清晰**

    job 分为 step, step 分为 reader, processor, writer, 增加了不同的 job 之间输入/处理/输出能力的复用性

    ![](/assets/2020101400.png)

1.2 **任务运行情况和统计数据清楚**

每次 job 的入参, 运行时间, 读取\(成功/失败\)处理\(失败/过滤\)写入\(成功/失败\)持久化表中, 便于排查问题

![](/assets/2020101401.png)



1.3 **配置性强\(跳过策略/重试策略,自定义监听器\)**

容错性强, 处理失败的数据可以通过观察者模式发送邮件/输出日志记录等

官方文档: https://spring.io/projects/spring-batch

## Spring Batch 入门例子

假设我们有一些拼单的商品\(**Commodity**\), 一些拼单的名媛\(**Girl**\), 我们需要批量读取这些商品, 把这些名媛和商品关联起来生成订单\(**GoodsOrder**\)并打印, 让我们看看这个过程应该怎么用 Spring Batch 实现吧!

### 定义 Reader

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

### 定义 Processor

```java
@Component
public class GirlProcessor implements ItemProcessor<Commodity, GoodsOrder> {

    @Override
    public GoodsOrder process(Commodity commodity) throws Exception {
        System.out.println(commodity);
        final ArrayList<Girl> girlList = new ArrayList<>();
        for (int i = 0; i < commodity.getBuyerCount(); i++) {
            girlList.add(new Girl("girl" + i));
        }
        return new GoodsOrder(commodity, girlList);
    }
}
```

### 定义 Writer

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
public void demo1() throws Exception {
    System.out.println("job starting");
    List<Commodity> commodityList = new ArrayList<>();
    for (int i = 0; i < 10; i++) {
        commodityList.add(new Commodity("酒店" + i, i));
    }
    final ListReader<Commodity> reader = new ListReader<>();
    reader.setSourceList(commodityList);
    final Job girlJob = jobBuilderFactory.get("demo1-girlsInShanghaiJob")
        .flow(stepBuilderFactory.get("demo1-girlsInShanghaiStep")
            .<Commodity, GoodsOrder>chunk(2).reader(reader)
            .processor(girlProcessor).writer(printWriter).build())
        .end().build();
    jobLauncher.run(girlJob, new JobParametersBuilder()
        .addDate("start_time",new Date()).toJobParameters());
}
```

### 输出:

```java
Commodity(name=酒店0, buyerCount=0)
Commodity(name=酒店1, buyerCount=1)
GoodsOrder(commodity=Commodity(name=酒店0, buyerCount=0), buyerList=[])
GoodsOrder(commodity=Commodity(name=酒店1, buyerCount=1), buyerList=[Girl(name=girl0)])
Commodity(name=酒店2, buyerCount=2)
Commodity(name=酒店3, buyerCount=3)
GoodsOrder(commodity=Commodity(name=酒店2, buyerCount=2), buyerList=[Girl(name=girl0), Girl(name=girl1)])
GoodsOrder(commodity=Commodity(name=酒店3, buyerCount=3), buyerList=[Girl(name=girl0), Girl(name=girl1), Girl(name=girl2)])
Commodity(name=酒店4, buyerCount=4)
Commodity(name=酒店5, buyerCount=5)
GoodsOrder(commodity=Commodity(name=酒店4, buyerCount=4), buyerList=[Girl(name=girl0), Girl(name=girl1), Girl(name=girl2), Girl(name=girl3)])
GoodsOrder(commodity=Commodity(name=酒店5, buyerCount=5), buyerList=[Girl(name=girl0), Girl(name=girl1), Girl(name=girl2), Girl(name=girl3), Girl(name=girl4)])
Commodity(name=酒店6, buyerCount=6)
Commodity(name=酒店7, buyerCount=7)
GoodsOrder(commodity=Commodity(name=酒店6, buyerCount=6), buyerList=[Girl(name=girl0), Girl(name=girl1), Girl(name=girl2), Girl(name=girl3), Girl(name=girl4), Girl(name=girl5)])
GoodsOrder(commodity=Commodity(name=酒店7, buyerCount=7), buyerList=[Girl(name=girl0), Girl(name=girl1), Girl(name=girl2), Girl(name=girl3), Girl(name=girl4), Girl(name=girl5), Girl(name=girl6)])
Commodity(name=酒店8, buyerCount=8)
Commodity(name=酒店9, buyerCount=9)
GoodsOrder(commodity=Commodity(name=酒店8, buyerCount=8), buyerList=[Girl(name=girl0), Girl(name=girl1), Girl(name=girl2), Girl(name=girl3), Girl(name=girl4), Girl(name=girl5), Girl(name=girl6), Girl(name=girl7)])
GoodsOrder(commodity=Commodity(name=酒店9, buyerCount=9), buyerList=[Girl(name=girl0), Girl(name=girl1), Girl(name=girl2), Girl(name=girl3), Girl(name=girl4), Girl(name=girl5), Girl(name=girl6), Girl(name=girl7), Girl(name=girl8)])
```

![](/assets/2020101402.png)

## 附录

可以运行的项目代码地址: https://github.com/currentperson/spring-codog 

本系列文章地址: https://github.com/currentperson/spring-study-notbook 

设计模式系列文章地址: https://github.com/currentperson/codog-java-design-pattern

设计模式项目代码地址: https://github.com/currentperson/CodogDesignPattern



