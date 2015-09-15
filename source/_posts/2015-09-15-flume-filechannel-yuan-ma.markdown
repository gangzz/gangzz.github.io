---
layout: post
title: "flume FileChannel 源码"
date: 2015-09-15 23:14:58 +0800
comments: true
categories: ["others","flume"]
---


![Flume结构](/images/blog/20150915/UserGuide_image00.png)

FileChannel是Flume Channel的一种基于文件系统的实现，能够直接持久化消息（FlumeEvent）到文件系统中。

## 参数&功能

FileChannel的参数包含但不限于以下几个主要参数

* checkpointDir：checkpoint数据的存储目录。
* useDualCheckpoints：在更新checkpoint前是否备份。
* backupCheckpointDir：检查点数据备份目录，只有上一个值为true时才有意义。
* dataDirs：数据存放目录，允许一个或多个逗号分隔的目录。
* checkpointInterval：checkpoint的执行周期，默认3秒，单位毫秒。
* maxFileSize：产生新文件的临界值，文件的最大长度。

## 在FS中的组织结构

以下将从效率、可靠性、事务、更新策略上介绍FileChannel设计及其文件系统的利用。

![文件数据组织结构](/images/blog/20150915/fs_structure.png)

### data Dirs

dataDirs虽然可以接受多个逗号分隔的目录，但是只有在不同的存储介质上时才有其实际意义。FileChannel会以轮询的方式写入各个目录，只有独立介质的情况下才能达到减轻寻址等机械动作导致的性能开销。

### 效率

FileChannel的核心数据包括Record（消息或操作）、元数据和checkpoints三部分（FlumeEventQueue的持久化数据），先介绍Record数据。

Record数据存储在log-n的文件中，其中n为变量随文件长度达到最大限制而递增，最佳的效率只能发生在顺序写入的情况下。FileChannel支持增加、删除和事务回滚操作（put、take、rollback），但是所有操作都以操作日志的形式追加到log-n文件中。

对log-n文件的写动作发生在LogFile.Writer中，每段数据的格式相同[flag][len][data]一位标志位、int长度的位数表示数据长度、数据。flag选值：OP_RECORD、OP_NOOP、OP_EOF（记录、操作、结尾）。

      // OP_RECORD + size + buffer
      int recordLength = 1 + (int)Serialization.SIZE_OF_INT + buffer.limit();

### 可靠性

Flume要保证即使进程崩溃了也能保证数据的可靠性。

首先，从log-n文件数据直接写入文件系统中正常情况下是可靠的，但是崩溃可以发生在任何时候，比如记录写到一半。

因此，FileChannel使用checkpoint的方式，检查周期由checkpointInterval指定，默认3秒。FileChannel在内存中（FlumeEventQueue，下面介绍）维护了队列中、事务中（未提交）的记录，该数据周期性地序列化到checkpointsDir目录中，同时每个log-n文件维护了对应的log-n.meta元数据文件，记录checkpoint发生是当前文件最新的record_id。在恢复过程中只要找到checkpoint和各文件对应的record-id就能尽力挽回损失了（record-id之后的数据无法保证完整性）。

还有一个小概率事件——写checkpoint时崩溃了，导致旧的checkpoint文件删除而新的不完整。该问题可以设置useDualCheckpoints（默认false），即在更新checkpoint文件前先对其备份。

### FlumeEventQueue
FileChannel通过FlumeEventQueue管理了记录位置和支持事务。

记录Record位置使用logFileId和offset确认，前者说明记录保存在哪一个log-n文件中，后者则是在文件中起始位置的偏移量。

事务问题，FileChannel为每个线程开启一个事务，只有事务被提交其操作才会对外可见。FlumeEventQueue通过Map结构隔离了各个未提交事务的操作（transactionId作为key），根据事务提交与否觉得该key对应的值加入还是抛弃——虽然record保存在了log-n文件中，但如果因回滚抛弃了该record的位置信息，其数据对外仍然不可见。

### Record位置和文件滚动（roll）

当log文件到达maxFileSize限制时需要产生新的文件继续记录日志，出于效率考虑不会把旧文件中未出队的数据复制到新的log文件中的，此时旧的日志文件中仍然存在record数据，因此转为只读文件，并且FileChannel允许每个log-n文件有若干个只读的RandomAccessFile的handler以支持多线程。

在每次文件滚动时会尝试删除无用的log-n文件，Id从N-2开始的文件会在本轮被标记、在下一轮roll的过程中删除，N为滚动产生的最新log id。

## 类图结构
![FileChannel类图](/images/blog/20150915/FileChannelClasses.png)

* Channel 接口封装。
* BasicChannelSemantics 提供了基于事务的Channel的底层框架，子类需要实现createTransaction方法。
* BasticTransactionSemantics 提供了事务的底层框架实现，子类需要实现doPut、doGet、doCommit、doRollback等方法。
* FileBackedTransaction FileChannel中事务的实现。
* Log FileChannel中数据的存储，包括log-n、meta和通过FlumeEventQueue持久化checkpoint数据。
* FlumeEventQueue checkpoint和事务的管理。
* LogFile.Writer log-n文件的读写封装。
* LogFile.MetaDataWriter log-n对应的元数据文件读写。
* LogFile.CachedFsUsableSpace 当前文件系统可用空间的监控，除了根据写入情况递减可用空间还会定时与FS实际情况同步。
* LogFile.RandomReader 和Writer对应，只读log-n文件的读取封装

## SpillableFileChannel方案

FileChannel在保证安全的情况下牺牲了较大的性能，SpillableFileChannel提供了折中方案。SpillableFileChannel允许设置内存Channel的大小，只有内存Channel溢出时才将数据写入到FileChannel中，在安全和性能两者之间提供了平衡性可调的方案。