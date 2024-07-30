## 平台测试

结论：均不行

- 虚拟卡：

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=NTBjNjVjOGJhNzk1MzAwOWZkNmY0OWMzOTY3NzE4ZTJfYVQzNzdLaEFnVzJRdUhKTUk3NHNlR2dTUWFMY0hKT2NfVG9rZW46TWlvS2Iwalc3b1JRc3B4TlprS2NXTm1YbjFmXzE3MjIzMTU0OTk6MTcyMjMxOTA5OV9WNA)

- 物理卡：

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=NmNjOWQ5M2U2NmYxN2I3YjQyYTZlYTIyODRkNzhjMjlfYjFYeUQ3WDBMMTVhY3Y3NktaQlNGMzVnUGlTS09Qak5fVG9rZW46VlFLM2JKZ2hqb1hnaUt4ZnlnVWNseVh1bm9kXzE3MjIzMTU0OTk6MTcyMjMxOTA5OV9WNA)

- CPU:

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=YjhkODVlNDFiMmJiODBkNDA2NzZmMDAwOTBlOGYxMTdfQmd3b01WUFFHZFB3VDBqRVRLODZsNjNLZDBtam1PWWZfVG9rZW46TTg4cWJQMVNEbzhrVWl4WDh3eGNUbVM5bmljXzE3MjIzMTU0OTk6MTcyMjMxOTA5OV9WNA)

报的这个GLIBC_PRIVATE错误，应该是镜像版本太旧，要重新编译glibc，不易解决。

## 物理机

IP：192.168.*.*

结论：成功submit

一些坑：

1. 提交任务时，可按照下面的demo设置好host，避免映射失败

```Python
import datetime
from pyspark.sql import SparkSession
from pyspark.sql.types import *
from pyspark.sql.functions import col

def time():
    return datetime.datetime.now()

start = time()

# 创建SparkSession
spark = SparkSession.builder \
    .appName("spark_demo") \
    .config("spark.master", "spark://VM-***-ubuntu:7077") \
    .config("spark.driver.host", "192.168.*.*") \
    .config("spark.driver.bindAddress", "192.168.3.***") \
    .getOrCreate()

sc = spark.sparkContext

# 建立一个rdd
data = [1, 2, 3, 4, 5, 6, 7, 8, 9]
rdd = sc.parallelize(data)

# 打印
print(rdd.collect())

# 停止SparkSession
spark.stop()
```

1. Which pyspark

spark安装目录的bin文件里有pysaprk，spark-submit等命令，

conda环境里，如果pip了pyspark，也会有一个pyspark...

默认情况下，在全局使用pyspark会默认使用conda环境里的那个

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=Yjk3ODc0ODI5M2YzZGUyYWVjYmQzZjRmZDBjNjM5NjJfZWRlZGxlWUc3Y3lYdmcwdTdYNFExZ0xDbjliSjV2ZDNfVG9rZW46VlhKbmJBMGF2b0o3Smt4Q3FYRGN1dEU4bkFkXzE3MjIzMTU0OTk6MTcyMjMxOTA5OV9WNA)

如果想使用spark安装目录里的，需要指定好路径

1. Pip install pyspark

安装与集群版本一致的pyspark

```Shell
pip install pyspark==3.3.3
```

1. java环境变量

需要正确配置java的环境变量，原始物理机的/etc/profile里的JAVA_HOME可能有误，需要检查

保证java版本与spark对应，这里用jdk11

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=MTQ5MGQ1ZTMyOTVmYTk0M2M1N2JkYTYwNzVhOGE2YWNfVmphUUtnN25WZTB6Y1ZHT2VYclFJeWxUdzdaTTlWUERfVG9rZW46QXJ5VWJqV0xCb3ZYTXB4djY0R2NEZHhibmlmXzE3MjIzMTU0OTk6MTcyMjMxOTA5OV9WNA)

1. 环境变量设置

客户端非集群内部，需要把环境变量设好，如果driver在本地，cluster 集群在云上，两者之间的带宽不适合大数据量的回传数据实验。仅用于小批量

```Shell
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
export PYSPARK_PYTHON=/mnt/haiqiang/laioncu_wukong_0517/4_step/face_pyspark/bin/python
export SPARK_HOME=/mnt/haiqiang/laioncu_wukong_0517/spark-3.3.3-bin-hadoop3
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin
export ZOOKEEPER_HOME=/usr/local/apache-zookeeper-3.8.3-bin
export PATH=$PATH:$ZOOKEEPER_HOME/bin
export SPARK_CLASSPATH=/mnt/haiqiang/laioncu_wukong_0517/spark/jars:$SPARK_CLASSPATH
export PATH=$SPARK_HOME/bin:$PATH
```

1. 连接clickhouse

当前版本(0.3.2)jdbc不支持Array，我尝试更换jdbc的jar包(0.6.0)，无果

妥协方案：读表时，把array列转成string，读完后再转回来

```Python
# from pyspark.sql.functions import split, col

# 读取 ClickHouse 数据
df = spark.read \
    .format("jdbc") \
    .option("url", "jdbc:clickhouse://10.9.*.*:8123/one_db") \
    .option("dbtable", "(SELECT id, arrayStringConcat(feature, ',') as feature_str FROM a_table) tmp") \
    .option("user", "default") \
    .load()
df = df.withColumn("feature", split(col("feature_str"), ",").cast("array<float>"))
```