##**`Hbase Java API`**

------------------------------------------

#####如何使用`Java`连接`HBase`数据库

Java连接`HBase`需要两个类

--------------------------------------------------

* `HBaseConfiguration`

* `ConnectionFactory`

  **`HBaseConfiguration`**

  * 创建`Configuration`对象

    *`Configuration config = HBaseConfiguration.create();`*

    //使用`create()`静态方法就可以得到`Configuration`对象

  **`ConnectionFactory`**

  * 创建`Connection`对象

    `Connection connection = ConnectionFactory.createConnection(config);`

    //`config`为前文的的配置对象

###`HBase2.X`创建表

  ```
    TableName tableName = TableName.valueOf("test");定义表名
    //TableDescriptor对象通过TableDescriptorBuilder构建
    TableDescriptorBuilder tableDescriptor = TableDescriptorBuilder.newBuilder(tableName)
    ColumnFamilyDescriptor family = ColumnFamilyDescriptorBuilder.newBuilder(Bytes.toBytes("data")).build;
    //构建列簇对象
    tableDescriptor.setColumnFamily(family);
    //设置列簇
    admin.createTable(tableDescriptor.build());
    //创建表
  ```

####添加数据

```
  Table table = connection.getTable(tableName);
  //获取Table
  try{
    byte[] row = Bytes.toBytes("row1);
    //定义行
    Put put = new Put(row);
    //创建Put对象
    byte[] columnFamily = Bytes.toBytes("data");
    //列
    byte[] qualifter = Bytes.toBytes(String.valueOf(1));
    //列簇修饰词
    byte[] value = Bytes.toBytes("张三丰");
    //值
    put.addColumn(columnFamily,qualifer,value);
    table.put(put);
    //向表中添加数据
  }finally{
    //使用完了要释放资源
    table.close();
  }
```

----
####获取指定行的数据
我们使用Get对象与Table对象就可以获取到列表中的数据了。
```
//获取数据
Get get = new Get(Bytes.toBytes("row1"));
//定义get对象
Result result = table.get(get);
//通过table对象获取数据
System.out.println("Result: " + result);
//很多时候我们只需要获取"值" 这里表示获取data:1列簇的值
byte[] valueBytes = result.getValue(Bytes.toBytes("data"),Bytes.toBytes("1"));
//获取到的字节数
将字节转换成字符串
String valueStr = new String(valueBytes,"utf-8");
System.out.println("value: " + valueStr);

```

####扫描表中的数据
只获取一行数据显然不能满足我们全部的需求，我们想过要获取表中所有的数据应该怎么操作呢？

`Scan、ResultScanner`对象就派上用场了，接下来我们看个示例
```
Scan scan = new Scan();
ResultScanner scanner = table.getScanner(scan);
try{
  for (Result scannerResult: scanner){
    System.out.println("Scan: " + scannerResult);
    byte[] row = scannerResult.getRow();
    System.out.println("rowName:" + new String(row,"utf-8"));
  }finally{
    scanner.close()
  }
}
```

-----
####删除表
和HBase shell的 操作一样，在Java中我们要删除表，需要先禁用他，然后再删除它。

```
TableName tableName = TableName.valueOf("test");
admin.disableTable(tableName);
//禁用表
admin.deleteTable(tableName)；
//删除表
```
----

####过滤器

   1. 过滤器根据行键、列族、列、时间戳等条件来对数据进行过滤
   2. 过滤器在服务端与 Scan、Get 一起使用来过滤掉不需要的数据以减少在服务端与客户端传输的数据量，所有的过滤器均继承自抽象类：
   `org.apache.hadoop.hbase。filter.Filter.java`

   **顺序执行：**

   1. reset(): 重置过滤器状态，用刚开始一下行数据的处理
   2. filterAllRemaining(): 是否过滤剩下的所有数据，即是否结束盖茨扫描，如PageFilter获取数据达到分页数
   3. filterRowKey(): 根据行键决定是否要过滤该行数据
   4. filterKeyValue(): 用来过滤不需要的列数、列
   5. transform(): 用来修改列值，如KeyOnlyFilter移除列值只返回行键
   6. filterRowCells(): 用来修改返回的列值列表，如SingleColumnValueExclueFiter过滤满足条件的列
   7. filterRow(): 最后决定是否需要过滤该行数据，如SingleColumnValueFilter通过是否存在过滤的条件列来判断是否需要过滤该行。

---
####**常用过滤器使用兼容性**
|过滤器 |Batch Skip | WhileMatch| FilterList | Get |Scan|
|:-|:-:|:-:|:-:|:-:|:-:|
|RowFilter |支持|  支持|  支持 | 支持 | 不支持  |支持|
|FamilyFilter|支持 | 支持  |支持|  支持|  支持 | 支持|
|QualifierFilter|支持 | 支持  |支持 | 支持 | 支持|  支持|
|ValueFilter|支持 | 支持 | 支持|  支持 | 支持 | 支持|
|SingleColumnValueFilter| 支持  |支持 | 支持 | 支持| 不支持| 支持|
|SingleColumnValueExcludeFilter |支持 |支持 |支持|支 |不支持|支持
|PrefixFilter|支持 | 不支持 |支持  |支持 | 不支持 | 支持|
|PageFilter|支持 | 不支持 |支持 | 支持 | 不支持 | 支持|
|KeyOnlyFilter|支持 | 支持  |支持  |支持  |支持 | 支持|
|FirstKeyOnlyFilter| 支持 | 支持 | 支持  |支持 | 支持|  支持|
|InclusiveStopFilter |支持 | 不支持| 支持 | 支持 | 不支持|  支持|
|TimestampsFilter|支持 | 支持  |支持 | 支持  |支持|  支持|
|SkipFilter|支持 | 条件 | 条件 | 支持 | 不支持 | 支持|
|WhileMatchFilter |支持  条件 | 条件|  支持 | 不支持 | 支持|
|FilterList|支持 | 条件 | 条件  |支持 | 支持  |支持|
|DependentColumnFilter |不支持 |支持  |支持 | 支持 | 支持 | 支持
|ColumnCountGetFilter |支持  |支持|  支持 | 支持|  支持 | 支持
|ColumnPaginationFilter |支持  |支持 | 支持 | 支持 | 支持 | 支持|
|ColumnPrefixFilter |支持 | 支持  |支持 | 支持 | 支持 | 支持|
|ColumnRangeFilter |支持 | 支持 | 支持 | 支持 | 支持 | 支持|

-----
####过滤器详解:

1. **KeyOnlyFilter**: 该过滤器使得查询结果只返回行键，主要是重写了transformCell方法的值全部替换为空。
2. **FirstKeyOnlyFilter**: 该过滤器使得查询结果只返回每行的第一个单元值，它通常与KeyOnlyFilter一起使用来执行高效的行统计操作。
3. **PrefixFilter**: 该过滤器用来匹配行键包含指定前缀的数据。这个过滤器所实现的功能其实也可以由RowFilter 结合 RegexComparator或者Scan 指定开始和结束行键来实现。
4. **RowFilter**: 该过滤器通过行键来匹配满足条件的数据行。例如,使用BinaryComparator可以查询出具有某个行键的数据行；通过比较运算符筛选出符合某一条件的多条数据。
5. **SingleColumnValueFilter**: 该过滤器类似于关系数据库的where条件语句，通过判断数据行指定的列限定符对应的值是否匹配指定的条件(如等于1001)来决定是否将该数据行返回。由于HBase支持动态模式，因此有些数据可能不包括SingleColumnValueFilter指定的列限定符，代码signleColumnValueFilter.setFilterIfMissing(true);可以用来决定查询结果是否返回这些不包含指定列限定符的数据行，默认false表示返回，true表示不返回。
6. **TimestanpsFilter**: 该过滤器可以用来过滤某个时间戳的数据，可以与Get和Scan一起使用。同时Get和Scan提供的setTimeStamp和setTimeRange方法可以实现同样的功能。
7. **ValueFilter**: 该过滤器使用单元格的值来过滤数据，只满足指定条件、指定值得单元格才会被返回。与SignleColumnValueFilter不同的是后者需要指定匹配的例限定符并且返回的是整行数据。
8. **WhileMatchFilter**: 同属于装饰或者包装器类型过滤器。两者的区别是当WhileMatchFilter包装的过滤器条件不被满足时，WhileMatchFilter即停止往下扫描，返回已经扫描到的数据，而SkipFilter则是当包装的过滤器条件满足时跳过该满足条件的数据行。
9. **FilterList**: FilterList也是一个包装类型的过滤器，可以用来包装一个有序的Filter列表。因为FilterList也继承自Filter,所以可以通过这个特性来构造一个有层次结构的嵌套的FilterList.
FilterList的构造函数涉及2个参数，一个是包装的过滤器列表，另一个是列表的操作关系，包括FilterList.Operator.MUST_PASS_ALL和FilterList.Operator.MUST_PASS_ONE,前者表示查询的结果数据行需要满足包装的过滤器列表的所有过滤条件，后者表示只需要满足包装的过滤器列表中任意一个过滤器的过滤条件。因此，对于操作关系MUST_PASS_ALL,只要发现某个过滤器过滤条件不被满足，查询扫描即会停止返回，而MUST_PASS_ONE则需要对过滤器列表里面的过滤器全部执行过滤流程，MUST_PASS_ALL效率稍高。

----

###Hbase性能调优
* **客户端调优**
  1. 设置客户端写入缓存
  2. 设置合适的扫描缓存
  3. 跳过WAL写入
  4. 设置重试次数与间隔
  5. 选用合适的过滤器

* **服务端优化**
  1. 建DDL优化
  2. 禁止分区自动拆分与压缩
  3. 开启机柜感知
  4. 开启Short Circuit Local Reads
  5. 开启补偿重试读
  6. JVM内存优化

---

