## 前言

![](/assets/2020120501.png)

## Spring Batch 预定义 Readers&Writers

Spring batch 提供了一些预定义的 reader 和 writer, 还有自己的生态, 所以可以很方便的找到合适的通用的 reader 和 writer, 如果不能满足再自己定义.

**官方自己的文档在这里:**

https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/appendix.html\#listOfReadersAndWriters

主要是讲一些 关系型数据库, 非关系型数据库, 消息队列, 文件, json, xml 等格式的 reader 和 writer

**还有大家会很常用的 mybatis 为 spring batch 开发的 reader 和 writer:**

http://mybatis.org/spring/batch.html

注意在使用 'MyBatisPagingItemReader' 的时候要注意  ORDER BY id LIMIT \#{\_skiprows}, \#{\_pagesize}, 不然会一直只读到第一页

使用 'MyBatisBatchItemWriter' 的时候要注意默认是每一行数据都需要被更新到, 不然就会报错, 可以通过配置 'assertUpdates

 ' 避免这种限制

**还有我们自己定义的一些典型的 reader:**

EmailReader, HttpReader, MaxIdReader 等, 都是为了解决不同的问题, 写一个 reader 的难度不是很高, 我会放在下面的 github 里大家一起讨论

https://github.com/currentperson/spring-codog





















