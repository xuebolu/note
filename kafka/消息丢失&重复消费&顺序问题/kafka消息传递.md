### 消息传递语义（message delivery semantic）

- at most once：最多一次。消息可能丢失也可能被处理，但最多只会被处理一次。
- at least once：至少一次。消息不会丢失，但可能被处理多次。可能重复，不会丢失。
- exactly once：精确传递一次。消息被处理且只会被处理一次。不丢失不重复就一次。

###Kafka 三次消息传递的过程：

- 生产者发消息给 Kafka Broker。
- Kafka Broker 消息同步和持久化
- Kafka Broker 将消息传递给消费者。

***消息丢失、重复消费、消费乱序主要围绕这三个过程讨论***

### Kafka的ack机制

- acks=0，producer不等待broker的响应，效率最高，但是消息很可能会丢。
- acks=1，leader broker收到消息后，不等待其他follower的响应，即返回ack。也可以理解为ack数为1。此时，如果follower还没有收到leader同步的消息leader就挂了，那么消息会丢失。按照上图中的例子，如果leader收到消息，成功写入PageCache后，会返回ack，此时producer认为消息发送成功。但此时，按照上图，数据还没有被同步到follower。如果此时leader断电，数据会丢失。
- acks=-1，leader broker收到消息后，挂起，等待所有ISR列表中的follower返回结果后，再返回ack。-1等效与all。这种配置下，只有leader写入数据到pagecache是不会返回ack的，还需要所有的ISR返回“成功”才会触发ack。如果此时断电，producer可以知道消息没有被发送成功，将会重新发送。如果在follower收到数据以后，成功返回ack，leader断电，数据将存在于原来的follower中。在重新选举以后，新的leader会持有该部分数据。数据从leader同步到follower，需要2步：
  - 数据从pageCache被刷盘到disk。因为只有disk中的数据才能被同步到replica。
  - 数据同步到replica，并且replica成功将数据写入PageCache。在producer得到ack后，哪怕是所有机器都停电，数据也至少会存在于leader的磁盘内。