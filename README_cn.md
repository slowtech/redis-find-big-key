# redis-find-big-key
Redis 大 key 分析工具主要分为两类：

**1. 离线分析**

基于 RDB 文件进行解析，常用工具是 redis-rdb-tools（https://github.com/sripathikrishnan/redis-rdb-tools）。

不过这个工具已近 5 年未更新，不支持 Redis 7，而且由于使用 Python 开发，解析速度较慢。

目前较为活跃的替代工具是 https://github.com/HDT3213/rdb ，该工具支持 Redis 7，并使用 Go 开发。

**2. 在线分析**

常用工具是 redis-cli，提供两种分析方式：

1. --bigkeys：Redis 3.0.0 引入，统计的是 key 中元素的数量。
2. --memkeys：Redis 6.0.0 引入，通过`MEMORY USAGE`命令统计 key 的内存占用。

这两种方式的优缺点如下：

- 离线分析：基于 RDB 文件进行解析，不会对线上实例产生影响，不足的是操作相对复杂，尤其是对于很多 Redis 云服务，由于禁用了 SYNC 命令，无法直接通过 `redis-cli --rdb <filename>` 下载 RDB 文件，只能手动从控制台下载。
- 在线分析：操作简单，只要有实例的访问权限，即可直接进行分析，不足的是分析过程中可能会对线上实例的性能产生一定影响。 

本文要介绍的工具（`redis-find-big-key`）也是一个在线分析工具，其实现思路与`redis-cli --memkeys`类似，但功能更为强大实用。主要体现在：

1. 支持 TOP N 功能

   该工具能够输出内存占用最多的前 N 个 key，而 redis-cli 只能输出每种类型中占用最多的单个 key。

2. 支持批量分析

   该工具能够同时分析多个 Redis 节点，特别是对于 Redis Cluster，启用集群模式（`-cluster-mode`）后，会自动分析每个分片。而 redis-cli 只能针对单个节点进行分析。

3. 自动选择从节点进行分析

   为了减少对实例性能的影响，工具会自动选择从节点进行分析，即使指定的是主节点的地址。只有在没有从节点时，才会选择主节点进行分析。而 redis-cli 只能分析主节点。

## 测试时间对比

测试环境：Redis 6.2.17，单实例，used_memory_human 为 9.75G，key 数量 100w，RDB 文件大小 3GB。

以下是上述工具在获取内存占用最多的 100 个大 key 时的耗时情况：

| 工具                           | 耗时      |
| ------------------------------ | --------- |
| redis-rdb-tools                | 25m38.68s |
| https://github.com/HDT3213/rdb | 50.68s    |
| redis-cli --memkeys            | 40.22s    |
| redis-find-big-key             | 29.12s    |

## 工具效果

```bash
# ./redis-find-big-key -addr 10.0.1.76:6379 -cluster-mode
Log file not specified, using default: /tmp/10.0.1.76:6379_20250222_043832.txt
Scanning keys from node: 10.0.1.76:6380 (slave)

Node: 10.0.1.76:6380
-------- Summary --------
Sampled 8 keys in the keyspace!
Total key length in bytes is 2.96 MB (avg len 379.43 KB)

Top biggest keys:
+------------------------------+--------+-----------+---------------------+
|             Key              |  Type  |   Size    | Number of elements  |
+------------------------------+--------+-----------+---------------------+
| mysortedset_20250222043729:1 |  zset  | 739.6 KB  |    8027 members     |
|   myhash_20250222043741:2    |  hash  | 648.12 KB |     9490 fields     |
| mysortedset_20250222043741:1 |  zset  | 536.44 KB |    5608 members     |
|    myset_20250222043729:1    |  set   | 399.66 KB |    8027 members     |
|    myset_20250222043741:1    |  set   | 328.36 KB |    5608 members     |
|   myhash_20250222043729:2    |  hash  | 222.65 KB |     3917 fields     |
|   mylist_20250222043729:1    |  list  | 160.54 KB |     8027 items      |
|    mykey_20250222043729:2    | string | 73 bytes  | 7 bytes (value len) |
+------------------------------+--------+-----------+---------------------+
Scanning keys from node: 10.0.1.202:6380 (slave)

Node: 10.0.1.202:6380
-------- Summary --------
Sampled 8 keys in the keyspace!
Total key length in bytes is 3.11 MB (avg len 398.23 KB)

Top biggest keys:
+------------------------------+--------+------------+---------------------+
|             Key              |  Type  |    Size    | Number of elements  |
+------------------------------+--------+------------+---------------------+
| mysortedset_20250222043741:2 |  zset  | 1020.13 KB |    9490 members     |
|    myset_20250222043741:2    |  set   | 588.81 KB  |    9490 members     |
|   myhash_20250222043729:1    |  hash  |  456.1 KB  |     8027 fields     |
| mysortedset_20250222043729:2 |  zset  |  404.5 KB  |    3917 members     |
|   myhash_20250222043741:1    |  hash  | 335.79 KB  |     5608 fields     |
|    myset_20250222043729:2    |  set   | 195.87 KB  |    3917 members     |
|   mylist_20250222043741:2    |  list  | 184.55 KB  |     9490 items      |
|    mykey_20250222043741:1    | string |  73 bytes  | 7 bytes (value len) |
+------------------------------+--------+------------+---------------------+
Scanning keys from node: 10.0.1.147:6380 (slave)

Node: 10.0.1.147:6380
-------- Summary --------
Sampled 4 keys in the keyspace!
Total key length in bytes is 192.9 KB (avg len 48.22 KB)

Top biggest keys:
+-------------------------+--------+-----------+---------------------+
|           Key           |  Type  |   Size    | Number of elements  |
+-------------------------+--------+-----------+---------------------+
| mylist_20250222043741:1 |  list  | 112.45 KB |     5608 items      |
| mylist_20250222043729:2 |  list  | 80.31 KB  |     3917 items      |
| mykey_20250222043729:1  | string | 73 bytes  | 7 bytes (value len) |
| mykey_20250222043741:2  | string | 73 bytes  | 7 bytes (value len) |
+-------------------------+--------+-----------+---------------------+
```

## 工具地址

项目地址：https://github.com/slowtech/redis-find-big-key

可直接下载二进制包，也可进行源码编译。

### 直接下载二进制包

```bash
# wget https://github.com/slowtech/redis-find-big-key/releases/download/v1.0.0/redis-find-big-key-linux-amd64.tar.gz
# tar xvf redis-find-big-key-linux-amd64.tar.gz 
```

解压后，会在当前目录生成一个名为`redis-find-big-key`的可执行文件。

### 源码编译

```bash
# wget https://github.com/slowtech/redis-find-big-key/archive/refs/tags/v1.0.0.tar.gz
# tar xvf v1.0.0.tar.gz 
# cd redis-find-big-key-1.0.0
# go build
```

编译完成后，会在当前目录生成一个名为`redis-find-big-key`的可执行文件。

## 参数解析

```bash
# ./redis-find-big-key --help
Usage of ./redis-find-big-key:
  -addr string
        Redis server address in the format <hostname>:<port>
  -cluster-mode
        Enable cluster mode to get keys from all shards in the Redis cluster
  -concurrency int
        Maximum number of nodes to process concurrently (default 1)
  -direct
        Perform operation on the specified node. If not specified, the operation will default to executing on the slave node
  -log-file string
        Log file for saving progress and intermediate result
  -master-yes
        Execute even if the Redis role is master
  -password string
        Redis password
  -samples uint
        Samples for memory usage (default 5)
  -skip-lazyfree-check
        Skip check lazyfree-lazy-expire
  -sleep float
        Sleep duration (in seconds) after processing each batch
  -tls
        Enable TLS for Redis connection
  -top int
        Maximum number of biggest keys to display (default 100)
```

各个参数的具体含义如下：

- -addr：指定 Redis 实例的地址，格式为`<hostname>:<port>`，例如 10.0.0.108:6379。注意，

  - 如果不启用集群模式（-cluster-mode），可以指定多个地址，地址之间用逗号分隔，例如 10.0.0.108:6379,10.0.0.108:6380。
  - 如果启用集群模式，只能指定一个地址，工具会自动发现集群中的其它节点。

- -cluster-mode：开启集群模式。工具会自动分析 Redis Cluster 中的每个分片，并优先选择从节点，只有在对应分片没有从节点时，才会选择主节点进行分析。

- -concurrency：设置并发度，默认值为 1，即逐个节点进行分析。如果要分析的节点比较多，可以增加并发度来提升分析速度。

- -direct：在 -addr 指定的节点上直接进行分析，这样会跳过自动选择从节点这个默认逻辑。

- -log-file：指定日志文件路径，用于记录分析过程中的进度信息和中间过程信息。不指定则默认是`/tmp/<firstNode>_<timestamp>.txt`，例如 /tmp/10.0.0.108:6379_20250218_125955.txt。

- -master-yes：如果待分析的节点中存在主节点（常见原因：从节点不存在；通过 -direct 参数指定要在主节点上分析），工具会提示以下错误：

  > Error: nodes 10.0.1.76:6379 are master. To execute, you must specify --master-yes

  如果确定可以在主节点上进行分析，可指定 -master-yes 跳过检测。

- -password：指定 Redis 实例的密码。

- -samples：设置`MEMORY USAGE key [SAMPLES count]`命令中的采样数量。对于包含多个元素的数据结构（如 LIST、SET、ZSET、HASH、STREAM 等），采样数量过低可能导致内存占用估算不准确，而过高则会增加计算时间和资源消耗。SAMPLES 不指定的话，默认为 5。

- -skip-lazyfree-check：如果是在主节点上进行分析，需要特别注意过期大 key。因为扫描操作会触发过期 key 的删除，如果未开启惰性删除（`lazyfree-lazy-expire`），删除操作将在主线程中执行，删除大 key 时可能会导致阻塞，从而影响正常的业务请求。

  因此，当工具在主节点上进行分析时，会自动检查节点是否启用了惰性删除。如果未启用，工具将提示以下错误并终止操作，以避免对线上业务造成影响：

  > Error: nodes 10.0.1.76:6379 are master and lazyfree-lazy-expire is set to 'no'. Scanning might trigger large key expiration, which could block the main thread. Please set lazyfree-lazy-expire to 'yes' for better performance. To skip this check, you must specify --skip-lazyfree-check

  在这种情况下，建议通过`CONFIG SET lazyfree-lazy-expire yes`命令开启惰性删除。

  如果确认没有过期大 key，想跳过检测，可指定 -skip-lazyfree-check。

- -sleep：设置每扫描完一批数据后的休眠时间。

- -tls：启用 TLS 连接。

- -top:  显示占用内存最多的 N 个 key。默认是 100。



## 常见用法

### 分析单个节点

```bash
./redis-find-big-key -addr 10.0.1.76:6379
Scanning keys from node: 10.0.1.202:6380 (slave)
```

注意，在上面的示例中，指定的节点和实际扫描的节点并不相同。这是因为 10.0.1.76:6379 是主节点，而该工具默认会选择从库进行分析。只有当指定的主节点没有从库时，工具才会直接扫描该主节点。



### 分析单个 Redis 集群

```bash
./redis-find-big-key -addr 10.0.1.76:6379 -cluster-mode
```

只需提供集群中任意一个节点的地址，工具会自动获取集群中其它节点的地址。同时，工具会优先选择从节点进行分析，只有在某个分片没有从节点时，才会选择该分片的主节点进行分析。



### 分析多个节点

```bash
./redis-find-big-key -addr 10.0.1.76:6379,10.0.1.202:6379,10.0.1.147:6379
```

节点之间是相互独立的，可以来自同一个集群，也可以来自不同的集群。注意，如果 -addr 参数指定了多个节点地址，则不能再使用 -cluster-mode 参数。



### 对主节点进行分析

如果需要对主节点进行分析，可指定主节点并使用`-direct`参数。

```bash
./redis-find-big-key -addr 10.0.1.76:6379 -direct -master-yes
```



## 注意事项

1.该工具仅适用于 Redis 4.0 及以上版本，因为`MEMORY USAGE`和`lazyfree-lazy-expire`是从 Redis 4.0 开始支持的。

2.同一个 key 在 redis-find-big-key 和 redis-cli 中显示的大小可能不一致，这是正常现象。原因在于，redis-find-big-key 默认选择从库进行分析，因此通常显示的是从库中的 key 大小，而  redis-cli 只能对主库进行分析，显示的是主库中的 key 大小。看下面这个示例。

```bash
# ./redis-find-big-key -addr 10.0.1.76:6379 -top 1
Scanning keys from node: 10.0.1.202:6380 (slave)
...
Top biggest keys:
+------------------------------+------+------------+--------------------+
|             Key              | Type |    Size    | Number of elements |
+------------------------------+------+------------+--------------------+
| mysortedset_20250222043741:2 | zset | 1020.13 KB |    9490 members    |
+------------------------------+------+------------+--------------------+

# redis-cli -h 10.0.1.76 -p 6379 -c MEMORY USAGE mysortedset_20250222043741:2
(integer) 1014242

# echo "scale=2; 1014242 / 1024" | bc
990.47
```

一个是 1020.13 KB，一个是 990.47 KB。

如果直接通过 redis-find-big-key 查看主库中该 key 的大小，结果与 redis-cli 完全一致：

```bash
# ./redis-find-big-key -addr 10.0.1.76:6379 -direct --master-yes -top 1 --skip-lazyfree-check
Scanning keys from node: 10.0.1.76:6379 (master)
...
Top biggest keys:
+------------------------------+------+-----------+--------------------+
|             Key              | Type |   Size    | Number of elements |
+------------------------------+------+-----------+--------------------+
| mysortedset_20250222043741:2 | zset | 990.47 KB |    9490 members    |
+------------------------------+------+-----------+--------------------+
```

## 实现原理

该工具是参考`redis-cli --memkeys`实现的。

实际上，无论是`redis-cli --bigkeys`还是`redis-cli --memkeys`，调用的都是`findBigKeys`函数，只不过传入的参数不一样。

```c
/* Find big keys */
if (config.bigkeys) {
    if (cliConnect(0) == REDIS_ERR) exit(1);
    findBigKeys(0, 0);
}

/* Find large keys */
if (config.memkeys) {
    if (cliConnect(0) == REDIS_ERR) exit(1);
    findBigKeys(1, config.memkeys_samples);
}
```

接下来，我们看一下这个函数的具体实现逻辑。

```c
static void findBigKeys(int memkeys, unsigned memkeys_samples) {
    ...
    // 通过 DBSIZE 命令获取 key 的总数量
    total_keys = getDbSize();

    /* Status message */
    printf("\n# Scanning the entire keyspace to find biggest keys as well as\n");
    printf("# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec\n");
    printf("# per 100 SCAN commands (not usually needed).\n\n");

    /* SCAN loop */
    do {
        /* Calculate approximate percentage completion */
        pct = 100 * (double)sampled/total_keys;
      
        // 通过 SCAN 命令扫描 key
        reply = sendScan(&it);
        scan_loops++;
        // 获取当前批次的 key 名称。
        keys  = reply->element[1];
        ...
        // 使用 pipeline 技术批量发送 TYPE 命令，获取每个 key 的类型
        getKeyTypes(types_dict, keys, types);
        // 使用 pipeline 技术批量发送相应命令获取每个 key 的大小
        getKeySizes(keys, types, sizes, memkeys, memkeys_samples);

        // 逐个处理 key，更新统计信息
        for(i=0;i<keys->elements;i++) {
            typeinfo *type = types[i];
            /* Skip keys that disappeared between SCAN and TYPE */
            if(!type)
                continue;

            type->totalsize += sizes[i]; // 累计每个类型 key 的总大小
            type->count++; // 累计每个类型 key 的数量
            totlen += keys->element[i]->len; // 累计 key 的长度
            sampled++; // 累计扫描的 key 的数量
            // 如果当前 key 的大小超过该类型的最大值，则会更新该类型的最大键大小，并打印统计信息。
            if(type->biggest<sizes[i]) {
                if (type->biggest_key)
                    sdsfree(type->biggest_key);
                type->biggest_key = sdscatrepr(sdsempty(), keys->element[i]->str, keys->element[i]->len);
                ...
                printf(
                   "[%05.2f%%] Biggest %-6s found so far '%s' with %llu %s\n",
                   pct, type->name, type->biggest_key, sizes[i],
                   !memkeys? type->sizeunit: "bytes");

                type->biggest = sizes[i];
            }

            // 每扫描 100 万个 key，还会输出当前进度和扫描的 key 数量。
            if(sampled % 1000000 == 0) {
                printf("[%05.2f%%] Sampled %llu keys so far\n", pct, sampled);
            }
        }

        // 如果设置了 interval，则每执行 100 次 SCAN 命令，都会 sleep 一段时间。
        if (config.interval && (scan_loops % 100) == 0) {
            usleep(config.interval);
        }

        freeReplyObject(reply);
    } while(force_cancel_loop == 0 && it != 0);
    .. 
    // 输出总的统计信息
    printf("\n-------- summary -------\n\n");
    if (force_cancel_loop) printf("[%05.2f%%] ", pct); // 如果循环被取消，则显示进度百分比
    printf("Sampled %llu keys in the keyspace!\n", sampled); // 打印已经扫描的 key 的数量
    printf("Total key length in bytes is %llu (avg len %.2f)\n\n",
       totlen, totlen ? (double)totlen/sampled : 0); // 打印 key 名的总长度及平均长度

    // 输出每种类型最大键的信息
    di = dictGetIterator(types_dict);
    while ((de = dictNext(di))) {
        typeinfo *type = dictGetVal(de);
        if(type->biggest_key) {
            printf("Biggest %6s found '%s' has %llu %s\n", type->name, type->biggest_key,
               type->biggest, !memkeys? type->sizeunit: "bytes");
        } // type->name 是 key 的类型名称，type->biggest_key 是最大键的名称
    } // type->biggest 是最大键的大小，!memkeys? type->sizeunit: "bytes" 是大小单位。
    ..
    // 输出每种类型的统计信息
    di = dictGetIterator(types_dict);
    while ((de = dictNext(di))) {
        typeinfo *type = dictGetVal(de);
        printf("%llu %ss with %llu %s (%05.2f%% of keys, avg size %.2f)\n",
           type->count, type->name, type->totalsize, !memkeys? type->sizeunit: "bytes",
           sampled ? 100 * (double)type->count/sampled : 0,
           type->count ? (double)type->totalsize/type->count : 0);
    } // sampled ? 100 * (double)type->count/sampled : 0 是当前类型的 key 的数量在总扫描的 key 数量中的百分比。
    ..
    exit(0);
}
```

该函数的实现逻辑如下：

1. 使用 DBSIZE 命令获取 Redis 数据库中的 key 总数。
2. 使用 SCAN 命令批量扫描 key，并获取当前批次的 key 名称。
3. 使用 pipeline 技术批量发送 TYPE 命令，获取每个 key 的类型。
4. 使用 pipeline 技术批量发送相应命令获取每个 key 的大小：
   - 若指定了 --bigkeys，根据 key 类型使用对应命令获取大小：STRLEN（string 类型）、LLEN（list 类型）、SCARD（set 类型）、HLEN（hash 类型）、ZCARD（zset 类型）、XLEN（stream 类型）。
   - 若指定了 --memkeys，使用 MEMORY USAGE 命令获取 key 的内存占用。
5. 逐个处理 key，更新统计信息：若某个 key 的大小超过该类型的最大值，则更新最大值并打印相关统计信息。

6. 输出总结信息，展示每种 key 类型的最大 key 及其相关统计数据。
