---
title: Kafka高性能措施
date: 2019-04-08 17:05:32
tags:
---

### 总体架构
Kafka 集群中仅仅包含了Broker， zookeeper两个组件。
Broker是整个Kafka集群的核心引擎，负责消息的存储转发，并对外提供服务。Kafka集群可以通过非常简单的增删Broker，实现整个集群的扩缩容。Kafka对外提供的基本单位是Topic，那么实现Topic级别的平行扩展能力，也就实现到了应用级的平行扩展能力。为了实现应用级的平行扩展能力，Kafka采用了对Topic进行分区的做法，通过对Topic进行分区让不同的分区落在不同的Broker上，从而利用到跟多Broker的能力，最终实现应用级的水平扩展。  

Zookeeper则在整个集群中负责存储一些配置信息，Broker信息，Topic信息等等元数据，并且承担了一部分协调选主的功能，可以将其理解为Kafka集群的配置管理中心。


### 高效使用磁盘
#### 顺序写磁盘
Kafka通过将Partition分为多个Segment，每个Segment对应一个物理文件，新消息来的时候添加到文件的尾部，通过删除这个文件的方式去删除Partition内的旧数据。通过这种方式避免了对文件的随机写操作。


#### 充分利用Page Cache(How)
使用Page Cache的好处如下
- I/O Scheduler对将连续的小块写组装成大块的物理写从而提高性能
- I/O Scheduler会尝试将一些写操作重新按照顺序排好，从而减少磁盘头的移动时间
- 葱粉利用所有空闲内存（非JVM内存）。如果使用应用层Cache，会增加GC负担
- 读操作(How)可直接在Page Cache内进行。如果消费和生产速度相当，甚至不需要通过物理磁盘（直接通过Page Cache）交换数据
- 如果进程重启，JVM内的Cache会失效，但Page Cache仍然可用

Broker收到数据之后，写磁盘只是将数据写入Page Cache，并不保证一定完全写入磁盘。在机器宕机的时候是通过Kafka层面的Replicas机制来保证数据可靠性的。  
Kafka也提供了```flush.messages```和```flush.ms```两个参数来将Page Cache中的数据强制Flush到磁盘，但Kafka并不建议使用

#### 支持多Disk Drive
Broker的```logs.dirs```配置项，允许配置多个文件夹。如果机器上由多个Disk Drive，可将不同的Disk挂载到不同的目录，然后将这些目录都配置到```log.dirs```里。Kafka会尽可能将不同Partition分配到不同的目录，也即不同的Disk上，从而充分利用了多Disk的优势。


#### 零拷贝
Linux 2.4+通过```sendfile```系统调用，提供了零拷贝。数据通过DMA拷贝到内核态buffer后，直接通过DMA拷贝到NIC Buffer，无需CPU拷贝。  
而Kafka使用JAVA NIO的FileChannel的```transferTo```和```transferFrom```方法，如果草操作系统提供零拷贝，这两个方法就会充分利用零拷贝的优势。
除了减少数据拷贝外，整个过程只有两次上下文切换，因此大大提高了性能。  


### 减少网络开销
#### 批处理
批处理是一种常用的用于提高I/O性能的方式。对Kafka而言，批处理既减少了网络传输的Overhead，又提高了写磁盘的效率。  
Kafka0.8.1及以前的Producer区分同步Producer和异步Producer。同步Producer的send方法主要分两种形式。一种是接受一个KeyedMessage作为参数，一次发送一条消息。另一种是接受一批KeyedMessage作为参数，一次性发送多条消息。而对于异步发送而言，无论是使用哪个send方法，实现上都不会立即将消息发送给Broker，而是先存到内部的队列中，直到消息条数达到阈值或者达到指定的Timeout才真正的将消息发送出去，从而实现了消息的批量发送。  
Kafka 0.8.2开始支持新的Producer API，将同步Producer和异步Producer结合。虽然从send接口来看，一次只能发送一个ProducerRecord，而不能像之前版本的send方法一样接受消息列表，但是send方法并非立即将消息发送出去，而是通过batch.size和linger.ms控制实际发送频率，从而实现批量发送。  
由于每次网络传输，除了传输消息本身以外，还要传输非常多的网络协议本身的一些内容（称为Overhead），所以将多条消息合并到一起传输，可有效减少网络传输的Overhead，进而提高了传输效率。

#### 数据压缩
Kafka从0.7开始支持数据压缩再传输给Broker。Broker接收到消息之后，并不直接解压缩，而是将消息以压缩后的形式持久化到磁盘。Consumer Fetch到数据后再解压缩。  
因此Kafka的压缩不仅减少了Producer到Broker的网络传输负载，同时也降低了Broker磁盘操作的负载，也降低了Consumer与Broker间的网络传输量。


#### 高效的序列化方式
Kafka消息的Key和Payload的类型可自定义，只需同时提供相应的序列化器和反序列化器即可。因此用户可以通过使用快速且紧凑的序列化-反序列化方式来减少实际网络传输和磁盘存储的数据规模，从而提高吞吐率。  
需要注意的是需要取得网络传输速率和序列化速度之间的一个平衡。
