1. **数字格式化补0**
    ```
    val part = 12
    val newPart = f"$part%20d".replaceAll(" ","0")
    ```
    结果：newPart: String = 00000000000000000012

---
2. **日期转换**
 * String-->Date-->Long
    ```
    import java.text.SimpleDateFormat
    import java.util.Date
    val timeString_1:String = "2018-07-09"
    val timeDate_2:Date = new SimpleDateFormat("yyyy-MM-dd").parse(timeString_2)
    val timeLog_3:Long = timeDate_3.gettime
    ```
    结果: timeLong_3: Long = 1531065600000
* convert Long-->Date
    ```
    val longTime_2: Long = 1613839669 //unit: second
    val timeDate_2: String = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(longTime_2 * 1000)
    ```
    结果: timeDate_2: String = 2021-02-21 00:47:49

---
* String --> Date
    ```
    val timeString_1: String = "2018-08-23 23:14:01"
    val timeDate_1: Date = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse(timeString_1)
    ```
    结果: timeDate_1: Thu Aug 23 23:14:01 CST 2018

---
3. **OLAP查询引擎比较**
    |   |hive|impala|presto|sparksql|hawq|clickhouse|greenplum|
    |:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
    |查询速度快(多表关联)|1|5|4|3|4|3|3|
    |查询速度快(单表查询)|1|3|4|3|3|5|3|
    |系统负载低|4|2|2|2|2|2|2|
    |连接数据源丰富程度|1|3|5|3|3|1|1|
    |支持的数据格式|5|4|5|5|5|3|3|
    |标准sql支持程度|4|4|5|3|5|3|5|
    |系统易用性|5|5|5|4|3|5|5|
    |社区活跃度|5|4|5|5|3|2|4|
    |自定义函数开发周期快|5|4|5|4|4|1|4|

---
4. **Elasticsearch 集成hbase实现**
* **Coprocessor**: 一个是EndPoint（类似关系型数据库的存储过程），用以加快特定查询的响应
* **Observer**: 一个就是Observer（类似关系型数据库的触发器）
    * Observer也分为几个类型，其中RegionObserver提供了一组表数据操作的钩子函数，覆盖了Get、Put、Scan、Delete等操作（通常有pre和post两种情况，表示在操作发生之前或发生之后），我们可以通过重载这些钩子函数，利用RegionServer实现特定的数据处理需求。
    基于RegionObserver的钩子函数，我们可以覆盖Put及Delete方法来实现Hbase和ES直接的数据同步

---
* **流程图**
    ```sequence
    participant Client
    participant ES
    participant Hbase
   
    Client -> ES:条件查询
    Note right of Client:条件查询
    Hbase -> ES: 同步数据
    Note left of Hbase: 同步数据
    ES -> Client: 返回符合条件的RowKey
    Note left of ES: 返回符合条件的RowKey
    Client -> Hbase: 根据rowkey返回数据
    Note left of Hbase: 根据rowkey返回数据

    Hbase -> Client: 返回结果集
    Note left of Hbase: 返回结果集
    ```

    * 数据进入HBase时，利用Observer同步进入ES索引库；
    * 客户端根据查询条件，利用ES提供的Java API对ES发起查询请求；
    * ES返回符合条件的RowKey；
    * 客户端再根据RowKey去HBase获取数据；
    * 最后HBase返回结果集。

    1. **Observer解决的问题**
        Hbase是一个分布式的存储体系，数据按照RowKey分成不同的Region,再分配给RegionServer管理。但是RegionServer只承担了存储的功能，如果Region能拥有一部分的计算能力，从而实现一个HBase框架上的MapReduce,那HBase的操作性能将进一步提升。正是为了解决这个一问题，HBase0.92版本后推出了Coprocessor--协处理器，一个工作在Master/RegionSercer中的框架，能运行用户的代码，从而灵活地完成分布式数据处理的任务。

         Coprocessor 包含2个组件，一个是EndPoint(类似关系型数据库的存储过程)，用以加快特定查询的响应，另一个就是Observer(类似关系型数据库的触发器)。Oberserver也分为几个类型，其中RegionObserver提供了一组表数据操作的钩子函数，涵盖了Get、Put、Scan、Delete等操作(通常有pre和post两种情况，表示在操作发生之前或发生之后)，我们可以通过重载这些钩子函数，利用RegionServer实现特定的数据处理需求。
    2. **应用场景**
        在同一个主机集群上同时建立了HBase集群和ElasticSearch集群，存储到HBase的数据必须实时同步到ElasticSearch.而HBase和ElasticSearch都没有更新的概念，我们的需求可以简化为2个步骤:
        * 当一个新的Put操作产生时，将Put数据转化为json,索引到ElasticSearch，并把RowKey作为新文档的ID
        * 当一个新的Delete操作产生时，获取Delete数据的RowKey,删除ElasticSearch中对应的ID
    3. **Java实现**
    Observer的Java实现并不复杂，只需要继承BaseRegionObserver的基类，并重载postPut和postDelete两个函数。考虑到未来HBase的写入会比较频繁，我们利用ElasticSearch的Bulk API做了有个缓冲池: 不是每次提交HBase数据都出发索引操作，而是积累到一定数量或者到达一定时间间隔才去批量操作，从而降低了RegionServer的网络I/O压力
    4. **Observer**
    
       Observer提供2中部署方式:  
        
        1.全局部署。把jar包的路径加入HBASE_CLASSPATH并且修改hbase-site.xml，这样Observer会对每一个表都生效。  
        
        2.单表部署。通过HBase Shell修改表结构，加入coprocessor信息。
        显然后一种更加灵活。通过HBase Shell安装Observer的详细步骤如下:
        * 把java项目打包为jar包，上传到HDFS的特定路径
        * 进入HBase Shell，disable你希望加载的表
        * 通过以下指令激活Observer：
        ```
        alter 'table_name', METHOD => 'table_att', 'coprocessor' => 'hdfs:///your/jar/path/on/hdfs|com.foo.bar|1001|es_cluster=elasticsearch,es_type=video,es_index=demo,es_port=9300,es_host=localhost'
        ```
        coprocessor对应的格式化|分隔，依次为:
        
        * jar包的HDFS路径
        * Observer的主类
        * 优先级(一般不用改)
        * 参数(对应Java项目的readConfiguration函数)
        新安装的coprocessor会自动生成名称: coprocessor + $ + 序号(可通过desc 'table_name'查看)
        因为一张表可能拥有多个coprocessor,卸载需要输入对应的coprocessor名称 ，比如：
         ```
         alter 'table_name', METHOD => 'table_att_unset', NAME=> 'coprocessor$1'
         ``` 
        * 进入HBase Shell, enable启动加载的表
        **注意**: HBase Observer的部署有个大坑：
            * 修改Java代码后，上传到HDFS的jar包文件必须和之前不一样，否则就算卸载掉原有的coprocessor再重新安装也不能生效 


5. **单表部署方式与hbase集成**
* 准实时同步数据方式
* 使用自定义代码，每次插入，更新，删除数据时，插入es索引
* 非实时同步数据方式
* 定时任务，每天晚上定时更新es索引
    1. 方案描述:
    ES+Hbase对接大致有两种方式，需要根据当前的业务场景做相应的选择
    2. 方案1:
    如果是对写入数据性能要求高的业务场景，那么一份数据先写到Hbase,然后再写到ES中，两个写入流程独立，这样可以达到性能最大。
    缺点：可能存在数据的不一致性
    3. 方案2：
    这也是目前网上比较流行的方案，使用hbase的协处理器监听数据在Hbase中的变动，实时的更新ES中的索引。
    缺点: 协处理器会影响Hbase的性能
    4. 装载协处理器实施步骤
    登录hbase shell
    #创建测试表
    `create 'gejx_test',cf`
    #停用测试表
    `disable 'gejx_test'`
    #表与协处理器建立关系，方法名称必须指定table att
    ```
    alter 'gejx_test' , METHOD =>'table_att','coprocessor'=>'/test/hbase/observer-1.5-SNAPSHOT.jar|com.tairanchina.csp.dmp.examples.HbaseDataSyncEsObserver|1073741823|es_index=test_index'
    ```
    #启用表
    `enable 'gejx_test'`
    #查看表信息
    `desc 'gejx_test'`


####测试数据:
```
put 'gejx_test,'2','cf:name','gjx1'
delete 'gejx_test', '2','cf:name'
```

查看日志要先在 HBase Master UI 界面下，确定数据存储在哪个节点上，再到相应的节点下面的 /var/log/hbase 下查看日志

tail -100f hbase-hbase-regionserver-bj--bigdata-test-v-52-136.xcm.cn.log

####卸载协处理器：
```
disable 'gejx_test'
alter 'gejx_test', METHOD => 'table_att_unset', NAME => 'coprocessor$1'
注意:方法名称必须table_att_unset
enable 'gejx_test'
```
####测试:
Hbase集成elasticsearch测试场景
|序列号|测试任务|bulkActions|byteSizeValue(M)|concurrentRequests|
|:-:|:-:|:-:|:-:|:-:|
|1|批量写入|1000|5|1|
|2|批量写入|2000|50|10|
|3|批量写入|5000|100|20|
|4|批量写入|10000|200|40|
|5|批量写入|20000|400|60|

