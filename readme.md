# Themis 

## 简介

Themis是基于google提出的[percolator](http://research.google.com/pubs/pub36726.html)算法，在HBase上实现跨行、跨表事务。
Themis以HBase行级别事务为基础，通过client端的协同完成跨行事务。Themis依赖[chronos](https://github.com/XiaoMi/chronos)提供的全局单调递增时钟服务，为事务全局定序，确保事务的ACID特性。Themis的实现利用了HBase coprocessor框架，不需要对HBase的代码做修改，在server端加载Themis coprocessor后即可服务。Themis提供与HBase类似的数据读写接口：put/delete/get/scan，经过了几个月的正确性验证和性能测试，性能与[percolator](http://research.google.com/pubs/pub36726.html)论文中报告的结果相近。

## Themis API使用示例
Themis API与HBase原生API相近，我们首先给出示例代码需要的常量：

    private static final byte[] TABLENAME = Bytes.toBytes("ThemisTable");
    private static final byte[] ROW = Bytes.toBytes("Row");
    private static final byte[] ANOTHER_ROW = Bytes.toBytes("AnotherRow");
    private static final byte[] FAMILY = Bytes.toBytes("ThemisCF");
    private static final byte[] QUALIFIER = Bytes.toBytes("Qualifier");
    private static final byte[] VALUE = Bytes.toBytes(10);
    private static Configuration conf;

### 跨行写

    HConnection connection = HConnectionManager.createConnection(conf);
    Transaction transaction = new Transaction(conf, connection);
    ThemisPut put = new ThemisPut(ROW).add(FAMILY, QUALIFIER, VALUE);
    transaction.put(TABLENAME, put);
    put = new ThemisPut(ANOTHER_ROW).add(FAMILY, QUALIFIER, VALUE);
    transaction.put(TABLENAME, put);
    transaction.commit();

如果transaction.commit()成功，会保证对于ROW和ANOTHER_ROW的修改同时成功，并且对于读同时可见；如果commit失败，会保证ROW和ANOTHER_ROW的修改都失败，对于读都不可见。
### Themis读

    Transaction transaction = new Transaction(conf, connection);
    ThemisGet get = new ThemisGet(ROW).addColumn(FAMILY, QUALIFIER);
    Result resultA = transaction.get(TABLENAME, get);
    get = new ThemisGet(ANOTHER_ROW).addColumn(FAMILY, QUALIFIER);
    Result resultB = transaction.get(TABLENAME, get);
    // ... 

对于跨行读，themis可以确保读取完整的事务。
更多示例代码参见：org.apache.hadoop.hbase.themis.example.Example.java

## 原理和实现
### Themis原理

Themis实现了[percolator](http://research.google.com/pubs/pub36726.html)算法，依赖全局时钟服务[chronos](https://github.com/XiaoMi/chronos)为事务定序。
#### Themis的写步骤为：
1.在用户写中选取一个column做为primaryColumn，其余的column为secondaryColumns。Themis会为primaryColumn和scondaryColumn构建对应的持久化锁(persistentLock)信息。
2.从chronos取全局时间prewriteTs，进行prewrite。对于每一个column，themis会使用对应的lockColumn保留persistentLock；prewrite阶段在没有写冲突的情况下，将数据和持久化锁分别写入对应的column和lockColumn。
3.prewrite成功后，从chronos取全局时间commitTs，对primaryColumn进行commit。对于每一个column，themis会使用对应的writeColumn存储提交信息，内容是事务的prewriteTs。primaryColumn的提交需要确保其persistentLock没有被删除的情况下删除persistentLock并写入commit信息。
4.primaryColumn提交成功后，开始提交secondaryColumn，动作是删除persistentLock并写入commit信息。

Themis是通过prewrite/commit两阶段写来完成事务。primaryColumn的commit成功后，事务整体成功，对读可见；否则事务整体失败，对读不可见。

#### Themis的读步骤为：
1.从chronos取一个startTs，首先从lockColumn读取数据判断是否有读写冲突。
2.如果没有读写冲突，读取timestamp < startTs的最新提交的事务。

Themis可以确保读取commitTs < startTs的所有提交事物，即数据库在startTs之前的snapshot。

#### Themis的冲突解决。Themis可能会遇到写写冲突和读写冲突。解决冲突的关键是利用存储在persistentLock中的时间戳，判断冲突事务是否过期。如果过期，根据冲突事务的primaryColumn是否提交，回滚或提交事务；否则，当前事务失败。

更多原理细节参考[percolator](http://research.google.com/pubs/pub36726.html)

### Themis实现

## Themis使用

### Themis服务端

### Themis客户端

## 测试

### 正确性验证
### 性能测试

## 将来的工作