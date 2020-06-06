#### sqoop 安装
  
  * 版本sqoop1和sqoop2（2个版本不兼容），生产环境一般用sqoop1(1.4.x)

* sqoop list命令
  
    * 列出所有数据库
    sqoop list-databases --username sqoopuser --password 123456 --connect jdbc:mysql://{$yourDBIpAddr}:3316/
 
    * 列出指定数据库下的所有表
    sqoop list-tables --username sqoopuser --password 123456 --connect jdbc:mysql://{$yourDBIpAddr}:3316/{$yourTableName}

* sqoop import命令
  
    *  sqoop import --username sqoopuser --password 123456 --connect jdbc:mysql://{$yourDBIpAddr}:3316/{$yourDBName} --query "select * from {$yourTableName} where \$CONDITIONS" --target-dir /tmp/jbw/sqoop_data/ --fields-terminated-by ',' --split-by id -m 1


* 案例
  #!/usr/bin/env bash
sqoop import \
        --connect jdbc:oracle:thin:@10.244.231.116:1521:cdsfc1 \
        --username mac3sfc \
        --password 'wx8888' \
        --table J_FULL_LOGS_TWENTY \
        --fields-terminated-by '\t' \
        --delete-target-dir \
        --num-mappers 1 \
        --hive-import \
        --hive-database default \
        --hive-table hive_test_hao

        备注：oracle连接一定要用sid不要用service_name

#### sqoop进阶

全量数据导入
就像名字起的那样，全量数据导入就是一次性将所有需要导入的数据，从关系型数据库一次性地导入到Hadoop中（可以是HDFS、Hive等）。全量导入形式使用场景为一次性离线分析场景。用sqoop import命令，具体如下：

* 全量数据导入
sqoop import \
 --connect jdbc:mysql://192.168.xxx.xxx:3316/testdb \
 --username root \
 --password 123456 \
 --query “select * from test_table where \$CONDITIONS” \
 --target-dir /user/root/person_all \ 
 --fields-terminated-by “,” \
 --hive-drop-import-delims \
 --null-string “\\N” \
 --null-non-string “\\N” \
 --split-by id \
 -m 6 \

    * 重要参数说明：

      参数	说明
      – query	SQL查询语句
      – target-dir	HDFS目标目录（确保目录不存在，否则会报错，因为Sqoop在导入数据至HDFS时会自己在HDFS上创建目录）
      –hive-drop-import- delims	删除数据中包含的Hive默认分隔符（^A, ^B, \n）
      –null-string	string类型空值的替换符（Hive中Null用\n表示）
      –null-non-string	非string类型空值的替换符
      –split-by	数据切片字段（int类型，m>1时必须指定）
      -m	Mapper任务数，默认为4
* 增量数据导入
    事实上，在生产环境中，系统可能会定期从与业务相关的关系型数据库向Hadoop导入数据，导入数仓后进行后续离线分析。故我们此时不可能再将所有数据重新导一遍，此时我们就需要增量数据导入这一模式了。

    增量数据导入分两种，一是基于递增列的增量数据导入（Append方式）。二是基于时间列的增量数据导入（LastModified方式）。

    * 1、Append方式

      举个栗子，有一个订单表，里面每个订单有一个唯一标识自增列ID，在关系型数据库中以主键形式存在。之前已经将id在0~5201314之间的编号的订单导入到Hadoop中了（这里为HDFS），现在一段时间后我们需要将近期产生的新的订单数据导入Hadoop中（这里为HDFS），以供后续数仓进行分析。此时我们只需要指定–incremental 参数为append，–last-value参数为5201314即可。表示只从id大于5201314后开始导入。

      Append方式的全量数据导入
      sqoop import \
        --connect jdbc:mysql://192.168.xxx.xxx:3316/testdb \
        --username root \
        --password 123456 \
        --query “select order_id, name from order_table where \$CONDITIONS” \
        --target-dir /user/root/orders_all \ 
        --split-by order_id \
        -m 6  \
        --incremental append \
        --check-column order_id \
        --last-value 5201314

      重要参数说明：

      参数	说明
      –incremental append	基于递增列的增量导入（将递增列值大于阈值的所有数据增量导入Hadoop）
      –check-column	递增列（int）
      –last-value	阈值（int）
    * 2、lastModify方式
      此方式要求原有表中有time字段，它能指定一个时间戳，让Sqoop把该时间戳之后的数据导入至Hadoop（这里为HDFS）。因为后续订单可能状态会变化，变化后time字段时间戳也会变化，此时Sqoop依然会将相同状态更改后的订单导入HDFS，当然我们可以指定merge-key参数为orser_id，表示将后续新的记录与原有记录合并。

      将时间列大于等于阈值的数据增量导入HDFS
      sqoop import \
        --connect jdbc:mysql://192.168.xxx.xxx:3316/testdb \
        --username root \
        --password transwarp \
        --query “select order_id, name from order_table where \$CONDITIONS” \
        --target-dir /user/root/order_all \ 
        --split-by id \
        -m 4  \
        --incremental lastmodified \
        --merge-key order_id \
        --check-column time \
        **remember this date !!!**
        --last-value “2014-11-09 21:00:00”  

      重要参数说明：

      参数	说明
      –incremental lastmodified	基于时间列的增量导入（将时间列大于等于阈值的所有数据增量导入Hadoop）
      –check-column	时间列（int）
      –last-value	阈值（int）
      –merge-key	合并列（主键，合并键值相同的记录）
      并发导入参数如何设置？
      我们知道通过 -m 参数能够设置导入数据的 map 任务数量，即指定了 -m 即表示导入方式为并发导入，这时我们必须同时指定 - -split-by 参数指定根据哪一列来实现哈希分片，从而将不同分片的数据分发到不同 map 任务上去跑，避免数据倾斜。

      重要Tip：

      生产环境中，为了防止主库被Sqoop抽崩，我们一般从备库中抽取数据。
      一般RDBMS的导出速度控制在60~80MB/s，每个 map 任务的处理速度5~10MB/s 估算，即 -m 参数一般设置4~8，表示启动 4~8 个map 任务并发抽取。

* sqoop导入HBase
   
    *  --hbase-table：通过指定--hbase-table参数值，指明将数据导入到HBase表中，而不是HDFS上的一个目录。输入表中的每一行将会被转换成一个HBase Put操作的输出表的一行。
       --hbase-row-key：你可以使用--hbase-row-key参数，手动的指定row key。默认的情况下，Sqoop会将split-by 列作为HBase rowkey列。如果没有指定split-by值，它将会试图识别关系表的关键字。

       如果源表是组合关键字，--hbase-row-key 参数后面值是用逗号分隔的组合关键字属性的列表，在这样种情况下，通过合并组合关键字属性的值来产生HBase的Row key，每个值之间使用下划线分隔开来。

       --column-family：必须指定--column-family参数，每一个输出列都会被放到同一个family列族中。  

        --hbase-create-table：如果HBase中的目标表和列族不存在，如果你使用该参数，Sqoop在运行任务的时候会根据HBase的默认配置，首先创建目标表和列族。

       注意一：当源表中是组合关键字的时候，必须手动指定--hbase-row-key参数，Sqoop才能将数据导入到HBase中，否则不行。
       注意二：如果HBase中的目标表和列族不存在，如果没加--hbase-create-table参数，Sqoop job将会报错误退出运行。所以你在将数据从源表导入到HBase之前，需要首先在HBase中创建目标表和其对应的列族。
      Sqoop目前会序列化所有的字段值，将值转换为字符串表示，然后向HBase中插入UTF-8编码的字符串值的二进制值。 
    *  以单关键字作为Rowkey导入 
        ```
        sqoop import --append --connect jdbc:oracle:thin:@219.216.110.120:1521:orcl --username TEST1 --password test1 --table TEST1 --columns age  --hbase-table test1 --hbase-row-key id --column-family personinfo
       ```
    * 手动指定Rowkey

        ```
        sqoop import --append --connect jdbc:oracle:thin:@219.216.110.120:1521:orcl --username TEST1 --password test1 --table TEST1 --columns id,name,sex,age --hbase-create-table --hbase-table test2 --hbase-row-key id,name --column-family personinfo
        ```

