---
title: 从Hash索引到LSM树（一）
date: 2020-06-26
categories:
 - Java
tags:
 - java
 - 数据库索引
 - LSM
author: 元闰子
---


## 前言

**数据库**算是软件应用系统中最常用的一类组件了，不管是一个庞大而复杂的电商系统，还是一个简单的个人博客，多多少少都会用到数据库，或是存储海量的数据，或是存储简单的状态信息。一般地，我们都喜欢将数据库划分为**关系型数据库**和**非关系型数据库（又称NoSQL数据库）**，前者的典型代表是MySQL数据库，后者的典型代表是HBase数据库。不管是关系型，还是非关系型，数据库都离不开两个最基本的功能：（1）**数据存储**；（2）**数据查询**。简单来说就是，**当你把数据丢给数据库时，它能够保持下来，并在稍后你想获取的时候，把数据返回给你。**

围绕着这两个基本功能，各类数据库都运用了很多技术手段对其进行了优化，其中最广为人知的当属**数据库索引技术**。索引是一种数据结构，它在牺牲少量数据存储（写）性能的情况下，可以大幅提升数据查询（读）性能。索引也有很多种类型，**Hash索引**算是最简单高效的一种了，但是由于它自身的限制，在数据库系统中并不被广泛使用。当今最常用的索引技术是**B/B+树索引**，被广泛地应用在关系型数据库中，主要应用于**读多写少**的场景。随着NoSQL数据库的兴起，**LSM（Log-Structured Merged-Tree）树**也逐渐流行，并被Google的BigTable论文所发扬光大。严格来说，LSM树并不算一种传统意义上的索引，它更像是一种设计思想，主要应用于**写多读少**的场景。

本系列文章，将从实现最简单的Key-Value数据库讲起，然后针对实现过程中遇到的一些瓶颈，采用上述的索引技术，对数据库进行优化，以此达到对数据库的索引技术有一个较为深刻的理解。

## 最简单的数据库

Martin Kleppmann在《**Designing Data-Intensive Applications**》一书中给出了一个最简单数据库的实现：

```shell
#!/bin/bash
db_set() {
  echo "$1,$2" >> database
}

db_get() {
  grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

这不到10行的`shell`代码实现了一个简单的Key-Value数据库。它一共有两个函数，`db_set`和`db_get`，前者对应**数据存储**功能，后者对应**数据查询**功能。该数据库采用简单的文本格式（`database`文件）进行数据存储，每条记录包含了一个键值对，key和value之间通过逗号（`,`）进行分隔。

数据库的使用方法也很简单，通过调用`db_set key value`可以将key及其对应的value存储到数据库中；通过`db_get key`可以得到该key对应的value：

```shell
$ db_set 123456 '{"name":"London","attractions":["Big Ben","London Eye"]}' 
$ db_set 42 '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'
$ db_get 42
{"name":"San Francisco","attractions":["Golden Gate Bridge"]}
```

透过`db_set`的实现我们发现，该数据库每次写都是直接往`database`文件中追加记录，鉴于文件系统顺序写的高效，因此该数据库的数据写入具有较高的性能。但是追加写也意味着，对同一个key进行更新时，其对应的旧的value并不会被覆盖，这也使得每次调用`db_get`获取某个key的value时，总是需要遍历所有记录，找到所有符合条件的value，并取其中最新的一个。因此，该数据库的读性能是非常低的。

![最简单的数据库读写操作](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-10-12-155006.jpg)

下面我们采用Java对这个最简单的数据库进行重写：

```java
/**
 * SimpleKvDb.java
 * 追加写
 * 全文件扫描读
 */
public class SimpleKvDb implements KvDb {
    ...
    @Override
    public String get(String key) {
        try (Stream<String> lines = logFile.lines()) {
            // step1: 筛选出该key对应的的所有value（对应grep "^$1," database）
            List<String> values = lines
                    .filter(line -> Util.matchKey(key, line))
                    .collect(Collectors.toList());
            // step2: 选取最新值（对应 sed -e "s/^$1,//" | tail -n 1）
            String record = values.get(values.size() - 1);
            return Util.valueOf(record);
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
    }
    @Override
    public void set(String key, String value) {
        // step1: 追加写（对应 echo "$1,$2" >> database）
        String record = Util.composeRecord(key, value);
        logFile.append(record);
    }
    ...
}
```

使用JHM进行压测结果如下：

```java
Benchmark                                            Mode  Cnt  Score   Error  Units
SimpleKvDbReadBenchmark.simpleKvDb_get_10000_test    avgt    8  0.631 ± 0.033  ms/op // 读耗时，1w条记录
SimpleKvDbReadBenchmark.simpleKvDb_get_100000_test   avgt    8  7.020 ± 0.842  ms/op // 读耗时，10w条记录
SimpleKvDbReadBenchmark.simpleKvDb_get_1000000_test  avgt    8  62.562 ± 5.466 ms/op // 读耗时，100w条记录
SimpleKvDbWriteBenchmark.simpleKvDb_set_test         avgt    8  0.036 ± 0.005  ms/op // 写耗时
```

从结果可以看出，该数据库的实现具有较高的写性能，但是读性能很低，而且**读耗时会随着数据量的增加而线性增长**。

*那么如何优化`SimpleKvDb`的读性能？引入索引技术！*

索引（index）是从数据库中的数据衍生出来的一种数据结构，它并不会对数据造成影响，只会影响数据库的读写性能。对于读操作，它能快速定位到目的数据，从而极大地提升读性能；对于写操作，由于需要增加额外的更新索引操作，因此会略微降低写性能。

> 如前言所述，索引也有很多类型，每种索引的特点都各有差别。因此到底要不要采用索引，具体采用哪种索引，需要根据实际的应用场景来决定。

下面，我们将首先采用最简单高效的Hash索引对`SimpleKvDb`进行优化。

## 给数据库加上Hash索引

考虑到Key-Value数据库本身就类似于Hash表的这个特点，我们很容易想到如下的索引策略：**在内存中维护一个Hash表，记录每个key对应的记录在数据文件（如前文所说的`database`文件）中的字节位移（byte offset）**。

对于写操作，在往数据文件追加记录后，还需要更新Hash表；对于读操作，首先通过Hash表确定该key对应记录在数据文件中的位移，然后通过字节位移快速找到value在数据文件中的位置并读取，从而避免了全文遍历这样的低效行为。

![加上索引之后的读写操作](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-10-12-155036.jpg)

给`SimpleKvDb`加上Hash索引，对应的代码实现为：

```java
/**
 * HashIdxKvDb.java
 * 追加写
 * 使用Hash索引提升读性能
 */
public class HashIdxKvDb implements KvDb {
    // 数据存储文件
    private final LogFile curLog;
    // 索引，value为该key对应的数据在文件中的offset
    private final Map<String, Long> idx;
    ...
    @Override
    public String get(String key) {
        if (!idx.containsKey(key)) {
            return "";
        }
        // step1: 读取索引
        long offset = idx.get(key);
        // step2: 根据索引读取value
        String record = curLog.read(offset);
        return Util.valueOf(record);
    }
    @Override
    public void set(String key, String value) {
        String record = Util.composeRecord(key, value);
        long curSize = curLog.size();
        // step1: 追加写数据
        if (curLog.append(record) != 0) {
            // step2: 更新索引
            idx.put(key, curSize);
        }
    }
		...
}
```

在实现`HashIdxKvDb`之前，我们将数据存储文件抽象成了`LogFile`对象，它的两个基本方法是`append`（追加写记录）和`read`（根据offset读记录），其中，`read`函数通过`RandomAccessFile`的`seek`方法快速定位到记录所处的位置，具体实现如下：

```java
// 追加写的日志文件，存储数据库数据
class LogFile {
    // 文件所在路径
    private Path path;
		...
    // 向日志文件中写入一行记录，自动添加换行符
    // 返回成功写入的字节大小
    long append(String record) {
        try {
            record += System.lineSeparator();
            Files.write(path, record.getBytes(), StandardOpenOption.APPEND);
            return record.getBytes().length;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return 0;
    }
    // 读取offset为起始位置的一行
    String read(long offset) {
        try (RandomAccessFile file = new RandomAccessFile(path.toFile(), "r")) {
            file.seek(offset);
            return file.readLine();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "";
    }
    ...
}
```

使用JHM进行压测结果如下：

```java
Benchmark                                              Mode  Cnt  Score   Error  Units
HashIdxKvDbReadBenchmark.hashIdxKvDb_get_10000_test    avgt    8  0.021 ± 0.001  ms/op // 读耗时，1w条记录
HashIdxKvDbReadBenchmark.hashIdxKvDb_get_100000_test   avgt    8  0.021 ± 0.001  ms/op // 读耗时，10w条记录
HashIdxKvDbReadBenchmark.hashIdxKvDb_get_1000000_test  avgt    8  0.021 ± 0.001  ms/op // 读耗时，100w条记录
HashIdxKvDbWriteBenchmark.hashIdxKvDb_set_test         avgt    8  0.038 ± 0.005  ms/op // 写耗时
```

从压测结果可以看出，相比`SimpleKvDb`，`HashIdxKvDb`的读性能有了大幅的提升，而且读耗时不再随着数据量的增加而成线性增长；而且写性能并没有明显的下降。

*虽然Hash索引实现很简单，却是极其的高效，它只需要1次磁盘寻址（`seek`操作），加上1次磁盘I/O（`readLine`操作），就能将数据加载出来。如果数据之前已经加载到文件系统缓存里，甚至都不用磁盘I/O。*

## 数据合并——compact

到目前为止，不管是`SimpleKvDb`，还是`HashIdxKvDb`，写操作都是不断在一个文件中追加数据，这种存储方式，通常我们称之为**append-only log**。

*那么，如何才能避免append-only log无休止的一直扩张下去，直到磁盘空间不足呢？*

append-only log的一个显著特点是旧的记录不会被覆盖删除，但是这些数据往往是无用的，因为读取某个key的value时，数据库都是取其最新的值。因此，解决该问题的一个思路是把这些无用的记录清除掉：

（1）当往append-only log追加数据到达一定大小后，另外创建一个新的append-only log进行追加。在这种机制下，数据分散到多个append-only log文件中存储，我们称这些log文件为**segment file**。

（2）保证只有当前的current segment file是可读可写，old segment file只读不写。

![segment file机制](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-10-12-155109.jpg)

（3）对old segment file进行compact操作——只保留每个key对应的最新记录，把的老记录删除。

![单文件compact操作](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-10-12-155144.jpg)

compact操作往往是在后台线程中执行，数据库会将合并的结果写到一个新的compacted segment file中，这样在执行compact操作时，就不会影响从old segment file中读数据的逻辑。等到compact操作完成之后，再把old segment file删除，后续的读操作迁移到compacted segment file上。

现在，单个compacted segment file中的key都是唯一的，但是多个compacted segment file之间还是有可能存在重复的key，我们还能够更近一步，对多个compacted segment file再一次进行compact操作，这样数据量会再次减少。

![多文件compact操作](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-10-12-155233.jpg)

> 类似的compact操作可以一层层执行下去，比如，可以对level2 compacted segment file进行compact，生成level3 compacted segment file。但是并不是compact的层次越多越好,具体的compact策略需要结合实际的应用场景进行设计。

## 给HashIdxKvDb加上compact机制

下面，我们试着给`HashIdxKvDb`加上compact机制。由于在compact机制下，数据会分散在多个segment file中存储，因此之前的Hash索引机制不再适用，我们需要给**每个segment file都单独维护一份Hash索引**。当然，这样也比较容易实现，只需要维护一个Hash表，key为segment file，value为该segment file对应的Hash索引。

![多segment file下的hash索引](http://yrunz-1300638001.cos.ap-guangzhou.myqcloud.com/2023-10-12-155247.jpg)

对应的代码实现如下：

```java
// 多文件哈希索引实现
class MultiHashIdx {
    ...
    private Map<LogFile, Map<String, Long>> idxs;
    // 获得指定LogFile中，Key的索引
    long idxOf(LogFile file, String key) {
        if (!idxs.containsKey(file) || !idxs.get(file).containsKey(key)) {
            return -1;
        }
        return idxs.get(file).get(key);
    }

    // 添加指定LogFile中Key的索引
    void addIdx(LogFile file, String key, long offset) {
        idxs.putIfAbsent(file, new ConcurrentHashMap<>());
        idxs.get(file).put(key, offset);
    }
    ...
}
```

另外，我们还需要在`CompactionHashIdxKvDb`里分别维护一份old segment file、level1 compacted segment file和level2 compacted segment file集合，并通过`ScheduledExecutorService`定时对这些集合进行compact操作。

```java
/**
 * 追加写，当当前segemnt file到达一定大小后，追加到新到segment file上。并定时对旧的segment file进行compact。
 * 为每个segment file维持一个哈希索引，提升读性能
 * 支持单线程写，多线程读
 */
public class CompactionHashIdxKvDb implements KvDb {
    ...
    // 当前追加写的segment file路径
    private LogFile curLog;

    // 写old segment file集合，会定时对这些文件进行level1 compact合并
    private final Deque<LogFile> toCompact;
    // level1 compacted segment file集合，会定时对这些文件进行level2 compact合并
    private final Deque<LogFile> compactedLevel1;
    // level2 compacted segment file集合
    private final Deque<LogFile> compactedLevel2;
    // 多segment file哈希索引
    private final MultiHashIdx idx;

    // 进行compact的定时调度
    private final ScheduledExecutorService compactExecutor;
    ...
}
```

相比于`HashIdxKvDb`， `CompactionHashIdxKvDb` 在写入新的数据之前，需要判断当前文件大小是否写满，如果写满了，则需要创建新的`LogFile`进行追加，并将写满后的`LogFile`归档到`toCompact`队列中。

```java
@Override
public void set(String key, String value) {
    try {
        // 如果当前LogFile写满了，则放到toCompact队列中，并创建新的LogFile
        if (curLog.size() >= MAX_LOG_SIZE_BYTE) {
            String curPath = curLog.path();
            Map<String, Long> oldIdx = idx.allIdxOf(curLog);
            curLog.renameTo(curPath + "_" + toCompactNum.getAndIncrement());
            toCompact.addLast(curLog);
            // 创建新的文件后，索引也要更新
            idx.addAllIdx(curLog, oldIdx);
            curLog = LogFile.create(curPath);
            idx.cleanIdx(curLog);
        }

        String record = Util.composeRecord(key, value);
        long curSize = curLog.size();
        // 写成功则更新索引
        if (curLog.append(record) != 0) {
            idx.addIdx(curLog, key, curSize);
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

`CompactionHashIdxKvDb` 的读操作相对麻烦，因为数据被分散在多个segment file上，所以需要按照如下顺序完成数据查找，直到查询到为止：**当前追加的LogFile->toCompact队列->compactedLevel1队列->compactedLevel2队列**。因此，`CompactionHashIdxKvDb` 的读操作在极端情况下（所查询的数据存储在`comactedLevel2`队列上时）也会较为低效。

```java
@Override
public String get(String key) {
    // 第一步：从当前的LogFile查找
    if (idx.idxOf(curLog, key) != -1) {
        long offset = idx.idxOf(curLog, key);
        String record = curLog.read(offset);
        return Util.valueOf(record);
    }
    // 第二步：从toCompact中查找
    String record = find(key, toCompact);
    if (!record.isEmpty()) {
        return record;
    }
    // 第三步：从 compactedLevel1 中查找
    record = find(key, compactedLevel1);
    if (!record.isEmpty()) {
        return record;
    }
    // 第四步：从 compactedLevel2 中查找
    record = find(key, compactedLevel2);
    if (!record.isEmpty()) {
            return record;
    }
    return "";
}
```

对`toCompact`队列里的old segment file进行单文件*level1 compact*操作时，可以利用Hash索引。因为Hash索引上所对应的记录总是最新的，因此只需遍历Hash索引，将每个key对应的最新记录查询出来，写到新的level1 compacted segment file中即可。

```java
// 进行level1 compact，对单个old segment file合并
void compactLevel1() {
    while (!toCompact.isEmpty()) {
        // 创建新的level1 compacted segment file
        LogFile newLogFile = LogFile.create(curLog.path() + "_level1_" + level1Num.getAndIncrement());
        LogFile logFile = toCompact.getFirst();
        // 只保留每个key对应的最新的value
        idx.allIdxOf(logFile).forEach((key, offset) -> {
            String record = logFile.read(offset);
            long curSize = newLogFile.size();
            if (newLogFile.append(record) != 0) {
                idx.addIdx(newLogFile, key, curSize);
            }
        });
        // 写完后存储到compactedLevel1队列中，并删除toCompact中对应的文件
        compactedLevel1.addLast(newLogFile);
        toCompact.pollFirst();
        logFile.delete();
    }
}
```

对`compactedLevel1`队列进行多文件*level2 compact*的策略是：**将当前队列里所有的level1 compacted segment file合并成一个level2 compacted segment file**。具体步骤如下：

1、生成一份`compactedLevel1`队列的`snapshot`。目的是为了避免在*level2 compact*过程中，有新的level1 compacted segment file加入到队列造成影响。

2、对`snapshot`进行compact操作，按照从新到旧的顺序，将level1 compacted segment file中的记录写入新的level2 compacted segment file中，如果发现level2 compacted segment file已经存在该key，则跳过。

3、等完成任务后，再从`compactedLevel1`队列里删除已经合并过的level1 compacted segment file。

```java
// 进行level2 compact，针对compactedLevel1队列中所有的文件进行合并
void compactLevel2() {
    ...
    // 生成一份快照
    Deque<LogFile> snapshot = new LinkedList<>(compactedLevel1);
    if (snapshot.isEmpty()) {
        return;
    }
    int compactSize = snapshot.size();
    // level2的文件命名规则为：filename_level2_num
    LogFile newLogFile = LogFile.create(curLog.path() + "_level2_" + level2Num.getAndIncrement());
    while (!snapshot.isEmpty()) {
        // 从最新的level1 compacted segment file开始处理
        LogFile logFile = snapshot.pollLast();
        logFile.lines().forEach(record -> {
            String key = Util.keyOf(record);
            // 只有当前level2 compacted segment file中不存在的key才写入
            if (idx.idxOf(newLogFile, key) == -1) {
                // 写入成功后，更新索引
                long offset = newLogFile.size();
                if (newLogFile.append(record) != 0) {
                    idx.addIdx(newLogFile, key, offset);
                }
            }
        });
    }
    compactedLevel2.addLast(newLogFile);
    // 写入完成后，删除compactedLevel1队列中相应的文件
    while (compactSize > 0) {
        LogFile logFile = compactedLevel1.pollFirst();
        logFile.delete();
        compactSize--;
    }
    ...
}
```

## 总结

本文首先介绍了Martin Kleppmann给出的最简单的数据库，并使用Java语言对它进行了重新的实现，接着采用Hash索引和compact机制对其进行了一系列的优化。

`SimpleKvDb`采用append-only log的方式存储数据，**append-only log最主要的优点是具备很高的写入性能**（文件系统的顺序写比随机写要快很多）。追加写也意味着同一个key对应的旧记录不会被覆盖删除，因此在查询数据时，需要遍历整个文件，找到该key的所有记录，并选取最新的一个。因为涉及到全文遍历，因此`SimpleKvDb`的读性能非常低。

为了优化`SimpleKvDb`的读性能，我们实现了具有Hash索引的`HashIdxKvDb`。Hash索引是一个在内存中维护的Hash表，保存每个key对应的记录在文件中的offset。因此**每次数据查询只需要1次磁盘寻址，加上1次磁盘I/O**，非常高效。

为了解决append-only log一直扩张导致磁盘空间不足的问题，我们给`HashIdxKvDb`引入了compact机制，实现了`CompactionHashIdxKvDb`。**compact机制能够有效清理数据库中的无效的旧数据，从而减缓了磁盘空间的压力**。

*Hash索引虽然简单高效，但是有如下两个限制：*

1、**必须在内存中维护Hash索引**。如果选择在磁盘上实现Hash索引，那么将会带来大量的磁盘随机读写，导致性能的急剧下降。另外，随着compact机制的引入，数据被分散在多个segment file中存储，我们不得不为每个segment file维护一份Hash索引，这就导致Hash索引的内存占用量不断增加，给系统带来了很大的内存压力。

2、**区间查询非常低效**。比如，当需要查询数据库中范围在`[key0000, key9999]`之间的所有key时，必须遍历索引中所有的元素，然后找到符合要求的key。

针对Hash索引的这两个限制，要怎样进行优化呢？本系列的下一篇文章，我们将介绍另一种不存在这两个限制的索引——LSM树。
