### kafka
>生产者：设置ACKS=2 则为2个ISR收到消息后消息才算成功，这样保证消息不丢失。每个分区都有自己的leader,其他则为follower，涉及生产者调优发送时间间隔linger.ms（默认值为0，减少网络IO）和batch.size批次大小（默认为16K）  
>消费者：批次消费max.poll.records(默认值500，可以调整到1-2千)一次拉取多少条数据，batch.size批次数据大小（默认16K） max.poll.interval.ms(默认5分钟)消息消费处理时间，超过该时间会认为消费失败，进行消费者重平衡rebalance，earliest从最早开始消费，latest从最近开始消费
***
>消费组管理：Heartbeat请求：consumer需要定期给coordinator发送心跳来表明自己还活着  
>JoinGroup请求：成员请求加入组  
>SyncGroup请求：group leader把分配方案告诉组内所有成员  
>LeaveGroup请求：主动告诉coordinator我要离开consumer group  
>DescribeGroup请求：显示组的所有信息，包括成员信息，协议名称，分配方案，订阅信息等。通常该请求是给管理员使用
```
所有消费组成员发送JoinGroup请求给Cooordinator
```

