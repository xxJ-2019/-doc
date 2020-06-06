#### HIVE 安装
* 从 Hive 2.1 版本开始, 我们需要先运行 schematool 命令来执行初始化操作
  ```
  schematool -dbType mysql -initSchema
  ```
* 元数据存储，一般用mysql存储（mysql安装见相关文档）
* 在HDFS上创建相关的目录，具体的目录，请对照hive-site.xml文件
* hive配置关注点：
  * hive-env.sh
  
    ```
    HADOOP_HOME=/home/apps/hadoop-3.1.1
    export HIVE_CONF_DIR=/home/apps/apache-hive-2.3.7-bin/conf
    export HIVE_AUX_JARS_PATH=/home/apps/apache-hive-2.3.7-bin/lib
    ```
  * hive-site.xml
```
<property>
    <name>hive.metastore.uris</name>
    <value>thrift://master:9083</value>
</property>

<property>
    <name>hive.metastore.warehouse.dir</name>
    <value>hdfs://master:9000/user/hive/warehouse</value>
</property>

<property>
    <name>hive.querylog.location</name>
    <value>/user/hive/log/hadoop</value>
    <description>Location of Hive run time structured log file</description>
</property>
<property>  
    <name>hive.exec.scratchdir</name>  
    <value>/user/hive/tmp</value>  
</property>  
<property>
  <!--端口改为你自己的端口，这里是连接数据库中onhive数据库，没有的话后面创建即可-->
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://10.244.237.95:3306/onhive?createDatabaseIfNotExist=true</value>
  <description>JDBC connect string for a JDBC metastore</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
<!--最新版本连接MySQL的jar包 所有写com.mysql.cj.jdbc.Driver,如果是旧版本用com.mysql.jdbc.Driver-->
  <value>com.mysql.cj.jdbc.Driver</value>
  <description>Driver class name for a JDBC metastore</description>
</property>
<property>
  <!--连接MySQL的用户名-->
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>root</value>
  <description>username to use against metastore database</description>
</property>
<property>
  <!--连接MySQL的密码-->
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>Adsl.2020</value>
  <description>password to use against metastore database</description>
</property>
```
  * 将hive的lib库添加到hadoop_class路径里面
  ```
    export HADOOP_CLASSPATH=/home/apps/hadoop-3.1.1/lib/*
    export HADOOP_CLASSPATH=$HADOOP_CLASSPATH:/home/apps/apache-hive-2.3.7-bin/lib/*
  ```
  * sparksql访问hive表数据

    * hive-site.xml添加配置
    ```
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>hdfs://master:9000/user/hive/warehouse</value>
    </property>
    ```
        
    * 将hive-site.xml拷贝到spark的conf目录下
  
    * 启动元数据服务
    ```
    hive --service metastore &
    ```

    * 启动spark集群，就可以直接用sparksql访问hive表数据

