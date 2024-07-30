# 数据

我随机生成了一个50_000_000x14的二维列表，并insert到0.32的spark_test数据库里，表单为application_data，字段如下：

```Bash
applicationId String
name String
addressLine1 String
zip String
city String
state String
country String
originationCountryCode String
creationTimestamp String
approved UInt8
creditLimit Int32
eventTimestamp String
ruleId Int32
rulePass UInt8
```

# Task1

比较dask和spark读clickhouse数据库，简单返回表的样本数量。

由于dask不能像spark一样用jdbc读clickhouse的数据，我利用clickhouse_driver，将表单分块，使用dask的延时计算操作，使得任务的构建和执行是分开的。日志如下，可见dask在compute之后才执行全部读表和count操作。整体耗时80-90s左右。

```Bash
2024-07-26 10:13:23.479 | INFO     | __main__:<module>:43 - START: Creating ddf
2024-07-26 10:13:23.484 | INFO     | __main__:<module>:45 - FINISH: Creating ddf Done
2024-07-26 10:13:23.484 | INFO     | __main__:<module>:47 - START: Starting count
2024-07-26 10:14:54.806 | INFO     | __main__:<module>:49 - FINISH: Count is 50000000
```

相同的，公平起见，我利用pyspark类似的延迟计算概念通过“行动（action）”和“转换（transformation）”来实现。日志如下，整体耗时70s。

```Bash
2024-07-26 11:02:19.441 | INFO     | __main__:<module>:66 - START: set partition_offsets
2024-07-26 11:02:19.441 | INFO     | __main__:<module>:71 - FINISH: set partition_offsets
2024-07-26 11:02:19.442 | INFO     | __main__:<module>:73 - START: parallelize offsets
2024-07-26 11:02:19.868 | INFO     | __main__:<module>:75 - FINISH: parallelize offsets
2024-07-26 11:02:19.868 | INFO     | __main__:<module>:77 - START: mapPartition
2024-07-26 11:02:19.871 | INFO     | __main__:<module>:79 - FINISH: mapPartition
2024-07-26 11:02:19.872 | INFO     | __main__:<module>:82 - START: createDataFrame
2024-07-26 11:02:23.186 | INFO     | __main__:<module>:84 - FINISH: createDataFrame
2024-07-26 11:02:23.186 | INFO     | __main__:<module>:86 - START: count
2024-07-26 11:03:23.622 | INFO     | __main__:<module>:88 - FINISH: count
```

二者分区设置相同，具体如下：

```Bash
num_partitions=25
total_rows=50000000
rows_per_partition = total_rows / num_partitions
```

使用jdbc读数据会更快，大概耗时12s，具体如下：

```Bash
def read_clickhouse_to_spark(spark, query):
    df = spark.read \
        .format("jdbc") \
        .option("url", "jdbc:clickhouse://10.9.0.32:8123/spark_test") \
        .option("dbtable", query) \
        .option("user", "default") \
        .load()
    return df
2024-07-25 17:06:36.567 | INFO     | __main__:<module>:21 - START: Creating spark dataframe
2024-07-25 17:06:39.654 | INFO     | __main__:<module>:24 - FINISH: Spark dataframe created
2024-07-25 17:06:39.654 | INFO     | __main__:<module>:26 - START: Starting count
2024-07-25 17:06:48.236 | INFO     | __main__:<module>:28 - FINISH: Count is 50000000
```

# Task2

为排除读数据的干扰，我提前把数据缓存，然后单独统计各自的计算性能.

首先遍历10次count操作，然后计算approved列的sum，再过滤originationCountryCode=="CAN"的样本，结果如下：

dask总用时5.05s，pyspark总用时：9.48s

其中，filter操作，dask耗时3.55s，pyspark耗时1.39s

dask：

```Bash
2024-07-26 15:33:35.857 | INFO     | __main__:<module>:93 - ******The 0 time******
2024-07-26 15:33:35.858 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.002 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.003 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.14 seconds
2024-07-26 15:33:36.004 | INFO     | __main__:<module>:93 - ******The 1 time******
2024-07-26 15:33:36.004 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.138 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.139 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.13 seconds
2024-07-26 15:33:36.140 | INFO     | __main__:<module>:93 - ******The 2 time******
2024-07-26 15:33:36.140 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.268 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.269 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.13 seconds
2024-07-26 15:33:36.270 | INFO     | __main__:<module>:93 - ******The 3 time******
2024-07-26 15:33:36.270 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.399 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.400 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.13 seconds
2024-07-26 15:33:36.401 | INFO     | __main__:<module>:93 - ******The 4 time******
2024-07-26 15:33:36.401 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.523 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.524 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.12 seconds
2024-07-26 15:33:36.525 | INFO     | __main__:<module>:93 - ******The 5 time******
2024-07-26 15:33:36.526 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.649 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.650 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.12 seconds
2024-07-26 15:33:36.651 | INFO     | __main__:<module>:93 - ******The 6 time******
2024-07-26 15:33:36.651 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.775 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.776 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.12 seconds
2024-07-26 15:33:36.777 | INFO     | __main__:<module>:93 - ******The 7 time******
2024-07-26 15:33:36.777 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:36.983 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:36.984 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.21 seconds
2024-07-26 15:33:36.985 | INFO     | __main__:<module>:93 - ******The 8 time******
2024-07-26 15:33:36.985 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:37.110 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:37.111 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.12 seconds
2024-07-26 15:33:37.112 | INFO     | __main__:<module>:93 - ******The 9 time******
2024-07-26 15:33:37.113 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:33:37.240 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:33:37.241 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.13 seconds
2024-07-26 15:33:37.241 | INFO     | utils.clickhouse_operator:wrapper:130 - START: sum of approved
2024-07-26 15:33:37.350 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 24994467
2024-07-26 15:33:37.351 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: sum of approved - Time taken: 0.11 seconds
2024-07-26 15:33:37.351 | INFO     | utils.clickhouse_operator:wrapper:130 - START: filter country
2024-07-26 15:33:40.904 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 25003039
2024-07-26 15:33:40.906 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: filter country - Time taken: 3.55 seconds
2024-07-26 15:33:40.907 | INFO     | __main__:<module>:100 - Total time taken: 5.05 seconds
```

Spark

```Bash
2024-07-26 15:30:50.927 | INFO     | __main__:<module>:113 - ******The 0 time******
2024-07-26 15:30:50.928 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:51.764 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:51.764 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.84 seconds
2024-07-26 15:30:51.765 | INFO     | __main__:<module>:113 - ******The 1 time******
2024-07-26 15:30:51.765 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:52.507 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:52.508 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.74 seconds
2024-07-26 15:30:52.509 | INFO     | __main__:<module>:113 - ******The 2 time******
2024-07-26 15:30:52.509 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:53.311 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:53.312 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.80 seconds
2024-07-26 15:30:53.313 | INFO     | __main__:<module>:113 - ******The 3 time******
2024-07-26 15:30:53.313 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:54.112 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:54.112 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.80 seconds
2024-07-26 15:30:54.113 | INFO     | __main__:<module>:113 - ******The 4 time******
2024-07-26 15:30:54.114 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:54.677 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:54.678 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.56 seconds
2024-07-26 15:30:54.678 | INFO     | __main__:<module>:113 - ******The 5 time******
2024-07-26 15:30:54.679 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:55.415 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:55.415 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.74 seconds
2024-07-26 15:30:55.416 | INFO     | __main__:<module>:113 - ******The 6 time******
2024-07-26 15:30:55.417 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:55.844 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:55.845 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.43 seconds
2024-07-26 15:30:55.845 | INFO     | __main__:<module>:113 - ******The 7 time******
2024-07-26 15:30:55.846 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:56.512 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:56.513 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.67 seconds
2024-07-26 15:30:56.513 | INFO     | __main__:<module>:113 - ******The 8 time******
2024-07-26 15:30:56.514 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:57.334 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:57.335 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.82 seconds
2024-07-26 15:30:57.336 | INFO     | __main__:<module>:113 - ******The 9 time******
2024-07-26 15:30:57.336 | INFO     | utils.clickhouse_operator:wrapper:130 - START: count
2024-07-26 15:30:58.151 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 50000000
2024-07-26 15:30:58.151 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: count - Time taken: 0.81 seconds
2024-07-26 15:30:58.152 | INFO     | utils.clickhouse_operator:wrapper:130 - START: sum of approved
2024-07-26 15:30:59.017 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 24993992
2024-07-26 15:30:59.018 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: sum of approved - Time taken: 0.86 seconds
2024-07-26 15:30:59.018 | INFO     | utils.clickhouse_operator:wrapper:130 - START: filter data
2024-07-26 15:31:00.410 | INFO     | utils.clickhouse_operator:wrapper:134 - RES: 25004727
2024-07-26 15:31:00.411 | INFO     | utils.clickhouse_operator:wrapper:135 - FINISH: filter data - Time taken: 1.39 seconds
2024-07-26 15:31:00.412 | INFO     | __main__:<module>:120 - Total time taken: 9.48 seconds
```

# Task3

使用蒙特卡罗法计算pi

dask：3.68s，spark：6.09s

dask_log:

```Bash
2024-07-26 16:10:27.690 | INFO     | __main__:<module>:34 - Pi is roughly 3.140388
2024-07-26 16:10:27.693 | INFO     | __main__:<module>:35 - FINSH: pi - Time taken: 3.68 seconds
```

saprk_log:

```Bash
2024-07-26 16:10:53.427 | INFO     | __main__:<module>:32 - Start: pi
2024-07-26 16:10:59.517 | INFO     | __main__:<module>:41 - FINSH: pi - Time taken: 6.09 seconds
```

# 源代码

路径：shuaikun@192.168.3.238:/home/shuaikun/my_pro/dask_spark/my_vs

```Bash
.
├── debug.log
├── generate_data
│   └── data.py
├── readme.md
├── utils
│   ├── clickhouse_operator.py
│   └── __pycache__
│       └── clickhouse_operator.cpython-39.pyc
└── vs
    ├── dask_pi.py
    ├── dask_single.py
    ├── logs
    │   ├── dask_pi.log
    │   ├── dask_single.log
    │   ├── spark_pi.log
    │   ├── spark_single_driver.log
    │   └── spark_single.log
    ├── spark_pi.py
    ├── spark_single_driver.py
    └── spark_single_jdbc.py
```