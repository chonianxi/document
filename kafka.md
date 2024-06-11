### kafka
>生产者：设置ACKS=2 则为2个ISR收到消息后消息才算成功，这样保证消息不丢失。每个分区都有自己的leader,其他则为follower，涉及生产者调优发送时间间隔linger.ms（默认值为0，减少网络IO）和batch.size批次大小（默认为16K）  
>消费者：

