## 前言

最近蚂蚁金服变成了 '马已今服' 了, 我的朋友圈小伙伴们的表现有捶胸顿足的, 还有转发网抑云的我不配, 没这命这样的歌, 积极的动态也就是上市从来不是最终的目标, 下面我们通过 springbatch 的跳过策略来讲讲这个问题

## Spring Batch 可跳过性

因为我们处理数据的时候有不同的需求, 比如有 100 条数据, 处理到第 6 条的时候失败了, 整个  step 都终止了, 这个可能是大多数批处理任务不希望看到的, 大多数的批处理任务是希望当前处理失败的记录下来, 单据补偿, 大多数的数据先跑过去, 这时候就要配置跳过策略了

## Spring Batch 可跳过性例子

假如福报厂的各个公司都要上一遍市, 我们定义一下这个读, 处理和写的过程

读:

```java
@Data
@NoArgsConstructor
public class ListReader<T> implements ItemReader<T> {

    private List<T> sourceList;
    private int index = 0;

    public ListReader(List<T> sourceList) {
        this.sourceList = sourceList;
    }

    @Override
    public T read() throws Exception {
        return sourceList.size() > index ? sourceList.get(index++) : null;
    }
}
```

处理:

```java
@Component
public class FubaoProcessor implements ItemProcessor<Company,String> {

    @Override
    public String process(Company company) throws Exception {
        if(Objects.equals(company.getName(),"蚂蚁金服")) {
            throw new RuntimeException("暂缓上市");
        }
        if(Objects.equals(company.getName(),"盒马")) {
            return null;
        }
        return company.getName() + "上市成功! 恭喜员工财务自由";
    }
}
```

写:

```java
@Component
public class FubaoWriter implements ItemWriter<String> {

    @Override
    public void write(List<? extends String> list) throws Exception {
        list.stream().forEach(item -> {
            if (item.contains("大文娱")) {
                throw new NonSkippableWriteException("上市异常", new RuntimeException("异常异常"));
            }
            System.out.println(item);
        });
    }
}
```

拼装:

```java
final String JOB_NAME = "demo4Ali";
List<Company> commodityList = Arrays.asList(new Company("蚂蚁金服"),
    new Company("菜鸟网络"),
    new Company("阿里云"),
    new Company("盒马"),
    new Company("大文娱"),
    new Company("飞猪"),
    new Company("阿里健康")
    );
final ListReader<Company> reader = new ListReader<>(commodityList);
final Job girlJob = jobBuilderFactory.get(JOB_NAME)
    .flow(stepBuilderFactory.get(JOB_NAME)
        .<Company, String>chunk(2).reader(reader)
        .processor(fubaoProcessor).writer(fubaoWriter).faultTolerant()
        .skipPolicy(new AlwaysSkipItemSkipPolicy()).listener(new SkipListener<Company, String>() {
            @Override
            public void onSkipInRead(Throwable throwable) {
                System.out.println("onSkipInRead: " + throwable);
            }

            @Override
            public void onSkipInProcess(Company company, Throwable throwable) {
                System.out.println("onSkipInProcess: " + company.getName() + "暂缓上市");
            }

            @Override
            public void onSkipInWrite(String s, Throwable throwable) {
                System.out.println("onSkipInWrite: <del>" + s + "</del>上市失败");
            }
        }).build())
    .end().build();
jobLauncher.run(girlJob, new JobParametersBuilder()
    .addDate("start_time", new Date()).toJobParameters());
```

表现:

```java
菜鸟网络上市成功! 恭喜员工财务自由
onSkipInProcess: 蚂蚁金服暂缓上市
阿里云上市成功! 恭喜员工财务自由
飞猪上市成功! 恭喜员工财务自由
onSkipInWrite: <del>大文娱上市成功! 恭喜员工财务自由</del>上市失败
阿里健康上市成功! 恭喜员工财务自由
```

## Spring课后作业

1. 尝试自定义跳过策略替换掉 AlwaysSkipItemSkipPolicy



