## 前言

最近美国大选终于尘埃落定, 我们的懂王落选了, 这下应该没空封禁 tiktok 了, 还衍生除了不少段子, 两个 70 多岁的老人都为了争取一份工作这么拼, 我们还有什么理由不努力! 这次大选因为这个统计选票的下班了, 选票就一直统计了好几天, 相比之下我们这些什么ppt, 会议, 项目还要 996, 难道比美国大选还重要么

## Spring Batch 并行流

当一个人一个票一个票的统计当然比较慢了, 可以大致把票分成 4 份, 每份由单独的人统计, 最后加加和, 速度就快多了

## Spring Batch 并行流例子

先定义一个投票类:

```java
@Data
@AllArgsConstructor
public class Vote {
    private String name;
}
```

定义一个处理器:

```java
@Component
public class VoteProcessor implements ItemProcessor<Vote, String> {

    @Override
    public String process(Vote vote) throws Exception {
        return "投票给了" + vote.getName();
    }
}
```

定义一个分段读取 Reader:

```java
@Data
@NoArgsConstructor
public class ListStepReader<T> implements ItemReader<T> {

    private List<T> sourceList;
    private int index = 0;
    private int finalIndex = 1;

    public ListStepReader(List<T> sourceList) {
        this.sourceList = sourceList;
    }

    public ListStepReader(List<T> sourceList, int index, int finalIndex) {
        this.sourceList = sourceList;
        this.index = index;
        this.finalIndex = finalIndex;
    }

    @Override
    public T read() throws Exception {
        return (finalIndex > index) ? sourceList.get(index++) : null;
    }
}
```

定义 step:

```java
private Flow voteFlow(List<Vote> voteList, int i) {
    return new FlowBuilder<SimpleFlow>("splitVoteFlow" + i)
        .start(stepBuilderFactory.get("splitVoteStep" + i)
            .<Vote, String>chunk(2).reader(new ListStepReader<>(voteList, (i - 1) * 4, i * 4))
            .processor(voteProcessor).writer(printWriter).build())
        .build();
}
```

组合流:

```java
public Flow splitFlow(List<Vote> voteList) {
    Flow[] flows = new Flow[4];
    for (int i = 0; i < flows.length; i++) {
        flows[i] = voteFlow(voteList, i + 1);
    }
    return new FlowBuilder<SimpleFlow>("splitVoteFlow")
        .split(taskExecutor4Vote())
        .add(flows)
        .build();
}
```

拼装 job:

```java
final String JOB_NAME = "demo4Vote";
List<Vote> voteList = new ArrayList<>();
for (int i = 0; i < 16; i++) {
    voteList.add(new Vote(i % 2 == 0 ? "川建国" : "拜登"));
}
final Job job = jobBuilderFactory.get(JOB_NAME)
    .start(splitFlow(voteList)).end().build();
jobLauncher.run(job, new JobParametersBuilder()
    .addDate("start_time", new Date()).toJobParameters());
```

输出:

```java
2020-11-14 21:22:00.078  INFO 64505 --- [   scheduling-1] o.s.b.c.l.support.SimpleJobLauncher      : Job: [FlowJob: [name=demo4Vote]] launched with the following parameters: [{start_time=1605360120058}]
2020-11-14 21:22:00.101  INFO 64505 --- [batch_for_vote2] o.s.batch.core.job.SimpleStepHandler     : Executing step: [splitVoteStep2]
2020-11-14 21:22:00.101  INFO 64505 --- [batch_for_vote1] o.s.batch.core.job.SimpleStepHandler     : Executing step: [splitVoteStep1]
2020-11-14 21:22:00.101  INFO 64505 --- [batch_for_vote4] o.s.batch.core.job.SimpleStepHandler     : Executing step: [splitVoteStep4]
2020-11-14 21:22:00.102  INFO 64505 --- [batch_for_vote3] o.s.batch.core.job.SimpleStepHandler     : Executing step: [splitVoteStep3]
投票给了川建国
投票给了拜登
投票给了川建国
投票给了拜登
投票给了川建国
投票给了拜登
投票给了川建国
投票给了拜登
投票给了川建国
投票给了拜登
投票给了川建国
投票给了拜登
投票给了川建国
投票给了拜登
投票给了川建国
投票给了拜登
2020-11-14 21:22:00.118  INFO 64505 --- [batch_for_vote2] o.s.batch.core.step.AbstractStep         : Step: [splitVoteStep2] executed in 17ms
2020-11-14 21:22:00.118  INFO 64505 --- [batch_for_vote4] o.s.batch.core.step.AbstractStep         : Step: [splitVoteStep4] executed in 16ms
2020-11-14 21:22:00.118  INFO 64505 --- [batch_for_vote1] o.s.batch.core.step.AbstractStep         : Step: [splitVoteStep1] executed in 17ms
2020-11-14 21:22:00.118  INFO 64505 --- [batch_for_vote3] o.s.batch.core.step.AbstractStep         : Step: [splitVoteStep3] executed in 16ms
2020-11-14 21:22:00.122  INFO 64505 --- [   scheduling-1] o.s.b.c.l.support.SimpleJobLauncher      : Job: [FlowJob: [name=demo4Vote]] completed with the following parameters: [{start_time=1605360120058}] and the following status: [COMPLETED] in 33ms
2020-11-14 21:22:08.085  INFO 64505 --- [extShutdownHook] o.s.s.c.ThreadPoolTaskScheduler          : Shutting down ExecutorService 'taskScheduler'
```



讨论, 并行流链接

