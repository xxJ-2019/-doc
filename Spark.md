**import**
* 需要导入elasticsearch-spark包
* 连接Elasticserach，创建索引，插入数据。

```
 val conf = new SparkConf().setAppName("appName").setMaster("local[2]")
    conf.set("es.index.auto.create", "true")
    conf.set("es.nodes", "10.244.237.96")
    conf.set("es.port", "9200")
    conf.set("es.net.http.auth.user", "elastic")
    conf.set("es.net.http.auth.pass", "Adsl2019")
    val sc = new SparkContext(conf)
    //    case class Trip(departure: String, arrival: String)
    //
    //    val upcomingTrip = Trip("OTP", "SFO")
    //    val lastWeekTrip = Trip("MUC", "OTP")
    //
    //    val rdd = sc.makeRDD(Seq(upcomingTrip, lastWeekTrip))
    //    EsSpark.saveToEs(rdd, "spark/docs")
    val json1 = """{"reason" : "business", "airport" : "SFO"}"""
    val json2 = """{"participants" : 5, "airport" : "OTP"}"""

    sc.makeRDD(Seq(json1, json2))
      .saveJsonToEs("spark/json-trips")
```