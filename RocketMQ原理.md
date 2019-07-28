# RocketMQ原理

![img](http://static.iocoder.cn/images/RocketMQ/2017_05_15/02.png)

### Producer发送定时消息

发送时设置**延迟级别**

### Broker存储定时消息

###### CommitLog#putMessage

存储消息时，将消息REAL_TOPIC和REAL_QUEUE_ID摘掉并存储

将此消息的topic更换为SCHEDULE_TOPIC_XXXX，queue_id则根据消息的延时等级进入对应的队列中

 

### Broker定时任务

![image-20190723103652554](/Users/jiangrui/Library/Application Support/typora-user-images/image-20190723103652554.png)

##### Broker发送定时消息 

SCHEDULE_TOPIC_XXXX下的每条消费队列都有一个与延时等级对应的定时任务

ScheduleMessageService会启动一个Timer的守护线程，每个延时队列会有一个与其**延时级别对应的TimerTask**

每个TimerTask再根据具体情况**安排自己队列里下次TimerTask的执行时机**

![img](http://static.iocoder.cn/images/RocketMQ/2017_05_15/01.png)

##### Broker持久化定时消息的发送进度

