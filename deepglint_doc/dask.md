# 简介

Dask 是一个面向 Python 的并行计算框架，可以将计算任务扩展到多核和集群上。它提供了两类 API：高层的 DataFrame 和 Array 模拟了 pandas 和 NumPy 的 API，开箱即用；底层的基于计算图的 API 可以与很多 Python 包相结合。基于这两种 API，Dask 已经形成了一些生态，以应对越来越大的数据量和各种各样数据科学任务。

Dask 的核心思想是构建任务计算图（Task Graph），将一个大计算任务分解为任务（Task），每个任务调用那些单机的 Python 包（比如 pandas 和 NumPy）作为执行后端。

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=Nzk4ODZjMDc0Yjk1OWU5Nzc2YzQ3NDI3ZjdkNmJkZDBfQXl3bmZHQVdNRWRITEdmUzI5SmdVWHpwenNqcVlnS2tfVG9rZW46WGVINGJWcThCb0hHWmp4aVozaGN5ZVZLbm1iXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

# 安装

```Python
pip install dask[complete]
```

如果你想可视化计算图，请提前安装graphviz

```Python
sudo apt-get install graphviz
```

# Helloworld

dask是延时（Lazy）执行的，只有在compute后才会计算。

```Python
import dask

ddf = dask.datasets.timeseries()
ddf
```

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=YjI2NDE5NTRiYTk0NzRjOTEzNzcxM2JmN2Q0YWYyNDBfb0JNSWRHNUpoOUNiOXVGZU1Qb0tzd2ltMHpmcEdNdHFfVG9rZW46RlhHV2JiaURkb2N4QTZ4dHo5WWNqQWNrbjFmXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

```Python
ddf.compute()
```

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=MzE3Njk3N2ZhZTgzYjYwZDVlMWZkMGEwYTQ2NTljZmJfZW93eHNISFpyOGNrWFRoUVJKT0I3OENhVDRhZkpGandfVG9rZW46TVlxd2JpRktsbzF1aUt4bWpBcWNiYWRBbm52XzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

API与pandas类似。

```Python
ddf2 = ddf[ddf.y > 0]
ddf3 = ddf2.groupby("name").x.std()
ddf3.visualize("./view/ddf3", format="svg")
ddf3.compute()
```

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=N2QzZWVmOTlmOGNlYzAwNjMxMzdjZDI0N2U3MDdhZDhfeGR0Tk5Fem1kbnljdVNpSVh4VVVOcktjQkJveFl6dlRfVG9rZW46TmhRRmI0a0k2b3BrV1V4a1FwTGNrNE51bnlkXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

# 集群

一个 Dask 集群必须包含一个调度器（Scheduler）和多个工作节点（Worker）。用户通过客户端（Client）向调度器提交计算任务，调度器对任务进行分析，生成 Task Graph，并将 Task 分发到多个 Worker 上。每个 Worker 承担一小部分计算任务，Worker 之间也要互相通信，比如计算结果的归集等。

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=YjgwMDA3OGUyNmUzYjQ3MmNiNDE4YzgyNWU0MmVhYWVfOFZteVRxYjJKSXlIY3Q0UWhPcE12a0FDQWxpN2UxRmNfVG9rZW46QXVic2IwQWw4b0NXaXh4b2N6QmNTZFJkbm5nXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

## localcluster

（只是介绍，实际在本机执行dask时，默认会拉满你的CPU，一句很拗口的话：Dask 将在完全包含您的本地进程的线程池中运行，因此，不必import LocalCluster，除非你想可视化任务）

默认情况（不进行任何额外的设置），Dask 会启动一个本地的集群 **`LocalCluster`**，并使用客户端 **`Client`** 连接这个集群。

Dask 探测到本地资源情况，比如本地有 4 个 CPU 核心、16GB 内存，根据本地资源创建了一个 `LocalCluster`。这个 `LocalCluster` 有 4 个 Worker，每个 Worker 对应一个 CPU 核心。

Dask 同时提供了仪表盘（Dashboard）链接，可以在网页中查看集群和作业的具体信息。

使用 `Client` 连接这个 `LocalCluster`，连接之后，Dask 所有的计算将提交到这个 `LocalCluster` 上

```Python
from dask.distributed import LocalCluster, Client

cluster = LocalCluster()
client = Client(cluster)
cluster
```

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=YzBmN2RkYmQ4ZWM5M2JiOTk1YTU5ZWE0ZmFlMDgyNDBfdmV6UWdkc3RhNVBHdTRqMXROS0tuNkxuckVWbzhxc0NfVG9rZW46WmVWQmJ5RHVYb2tDZVN4SzVhQmNHUjBQblNlXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

可以查看webUI(IP:8787)，里面详细记录了关于集群任务的详细信息。

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=MjJmODBhZjJmM2UwZGJiMjg3NDY5ZTc0NzA1NTAxNTNfUEh6SDVSSXNaWXdBVU4xMU5IWk85WmFZSWNFUWhYSzlfVG9rZW46Um9jOGJ0S1pmb29HSFZ4TDdTYWNEaml6bnhkXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

## 多机器集群

当我们有更多的机器，即计算节点时，可以使用命令行在不同的计算节点上启动 Dask Scheduler 和 Dask Worker。且跨系统。

节点1：192.168.3.238(linux)，执行：

```Python
dask scheduler
```

会打印如下日志：

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=OGQ5M2FjZDlhMDRhMGQxYTRmYmRmNmM3ZDVjZmE2NTFfYkhLVmpXaGNjVXlncTdmNWlCbmFEaHJoR0YybGZ6S09fVG9rZW46QzBEUWIwbkRSb3ZYdXh4TlRKdWN5MllNbjNjXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

在其他节点，如：192.168.75.81(windows)，执行：

```Python
dask worker tcp://192.168.3.238:8786
```

scheduler的默认端口是8786，可以指定端口：

```Python
dask scheduler --port 9786
```

示例：

```Python
import dask
import dask.datasets
import dask.datasets
from dask.distributed import Client


if __name__ == "__main__":
    client = Client("192.168.3.238:8786")
    df = dask.datasets.timeseries()
    df2 = df[df.y > 0].groupby("name").x.std()
    res = df2.compute()
    print(res)
```

注意：要保证集群的各个节点环境一致，否则，我也不清楚会出什么问题，如我的238和81的python版本不一致，上来就warning了。

![img](https://deepglint.feishu.cn/space/api/box/stream/download/asynccode/?code=YjUxYTg4MjI5NzIwOWIzNTRjZWI2YTNlMGM4ZDkxOTVfeHZjNHBtSWFselBVeE9zV1NWUmM3NjFTMFJFV2c4TFBfVG9rZW46VWswTmJIWEdOb2VFVjZ4UUI4dGNCZjNsbkpOXzE3MjIzMTU1NzU6MTcyMjMxOTE3NV9WNA)

# 连接clickhouse

dask不支持直接连接clickhouse，需要借助clickhouse第三方库clickhouse_driver。

注意：在导入第三方库的时候，dask和clickhouse都会import Client，请重命名。

示例：

```Python
import dask.dataframe as dd
from dask.distributed import LocalCluster, Client
import pandas as pd
from clickhouse_driver import Client as Client_click


def read_clickhouse_to_dask(host, database, table, user='default', password=''):
    # 创建 ClickHouse 连接
    client = Client_click(host=host, user=user, password=password, database=database)
    
    # 查询数据
    query = f'SELECT * FROM {table}'
    data = client.execute(query, with_column_types=True)
    # 获取列名和数据类型
    columns = [col[0] for col in data[1]]
    df = pd.DataFrame(data[0], columns=columns)
    
    # 将数据转换为 Dask DataFrame
    ddf = dd.from_pandas(df, npartitions=3)
    
    return ddf
    
    
if __name__ == "__main__":
    cluster = LocalCluster()
    client = Client(cluster)

    # 使用示例
    host = '10.9.0.32'
    database = 'spark_test'
    table = 'random_array'

    ddf = read_clickhouse_to_dask(host, database, table, index_col)
    print(ddf.head())
```