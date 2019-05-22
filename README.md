
# 分布式事务--基于可靠消息的最终一致性解决方案demo

![Spring Boot 2.0](https://img.shields.io/badge/Spring%20Boot-2.0-brightgreen.svg)
![Mysql 5.6](https://img.shields.io/badge/Mysql-5.6-blue.svg)
![JDK 1.8](https://img.shields.io/badge/JDK-1.8-brightgreen.svg)
![Maven](https://img.shields.io/badge/Maven-3.3.9-yellowgreen.svg)


## 项目使用技术

springboot、dubbo、zookeeper、activeMQ、定时任务

## 一、项目结构

maven父子工程：

父工程：consis

子工程：api-service、order、product、message

api-service:该项目主要是提供接口调用的，还包含实体类、枚举等一些通用内容

order:该项目是专门处理订单相关操作的系统

product:该项目是专门处理产品相关操作的系统

message:该项目是提供消息服务的系统，好包括定时任务


它们的依赖关系如下图：

![enter image description here](https://images.gitbook.cn/a181d460-acf7-11e8-afe5-6ba901a27e1b)

## 二、启动项目

首先在不同的系统中建立自己对应的数据库，springdatajpa会自动帮你生成表


为message工程创建transaction数据库(空库即可)


为product工程和order工程创建demo数据库(空库即可)


并在demo数据库中初始化2条product测试数据：

insert into `product` (`id`, `create_time`, `product_name`, `product_price`, `product_sku`, `product_spec`, `remark`) values('101','2018-01-10 12:32:05','苹果手机保护套','8888.00','516','保护套','');


insert into `product` (`id`, `create_time`, `product_name`, `product_price`, `product_sku`, `product_spec`, `remark`) values('102','2018-01-10 12:32:05','苹果手机XR MAX','125.00','115','苹果手机','苹果手机XR MAX');


注意启动顺序，如下：

1 启动zookeeper

2 启动activemq

3 启动tomcat，访问dubbo-admin管理控制台

4 启动MessageApplication
    定时重发消息
    定时将死亡的消息通知给工作人员，进行人工补偿操作
    
5 启动ProductApplication
    从mq消息中间件中监听并消费消息，将json消息转为订单对象
    根据消息编号查询该消息是否已被消费，保证幂等性
    如果消息未被消费（即存在此消息），则产品表扣减库存；如果已经消费（不存在此消息），则不做处理
    产品表扣减库存成功，则删除此消息，如果待处理消息日志表中有此消息，则更改状态为1，表示已处理；扣减失败，则不做处理

6 启动OrderApplication
    插入订单表之前，首先创建预发送消息，保存到事务消息表中，此时消息状态为：未发送
    插入订单，如果插入订单失败则将事务消息表中预发送消息删除
    插入订单成功后，修改消息表预发送消息状态为发送中，并发送消息至mq
    如果发送消息失败，则订单回滚并删除事务消息表消息

7. 启动成功后，就可以通过postman工具进行测试，如访问：http://localhost:8082/createOrder

{
	"buyerId": 10029122,
	"list": [{
			"id": "101",
			"num": 2,
			"product": {
				"id": 101,
				"productName": "苹果手机保护套",
				"productPrice": 125,
				"productSku": 524,
				"productSpec": "保护套",
				"createTime": "2019-05-22 12:00:00",
				"remark": "高大上的保护套"
			}
		},
		{
			"id": "102",
			"num": 2,
			"product": {
				"id": 102,
				"productName": "苹果手机XR MAX",
				"productPrice": 8888,
				"productSku": 123,
				"productSpec": "苹果顶配手机",
				"createTime": "2019-05-22 12:00:00",
				"remark": "高大上的手机"
			}
		}
	]
}
![postman调用示例图](https://github.com/marcusfang/consis/blob/master/postman-sample.jpg)

8. 分别观察这三个工程在控制台的日志在调用前后的变化：


如message工程的日志：
Hibernate: update transaction_message set create_time=?, dead_message=?, message_body=?, message_id=?, modify_time=?, queue_name=?, remark=?, send_times=?, status=? where id=?
2019-05-22 13:00:25.723  INFO 11492 --- [20880-thread-24] c.d.m.s.a.ActiveMqQueueServiceImpl       : >>> 发送消息到mq 队列:ORDER_QUEUE,消息内容:{"buyerId":10029122,"createTime":1558501225678,"list":[{"id":101,"num":2},{"id":102,"num":2}],"messageId":"30c6a800-5f8a-411d-b85b-f919e4980aff","orderCode":"9e8634bf-7670-429b-932d-eb2084930910","orderPrice":18026.00}
2019-05-22 13:00:25.723  INFO 11492 --- [20880-thread-24] c.d.m.s.i.TransactionMessageServiceImpl  : >>> 确认消息并且发送到消息中间件! 消息编号:30c6a800-5f8a-411d-b85b-f919e4980aff,队列名称:ORDER_QUEUE

如product工程的日志：
2019-05-22 12:57:38.425  INFO 15308 --- [enerContainer-1] c.d.p.s.impl.ProductConsumerListener     : >>> 业务执行成功删除消息! messageId:7d36a63e-6ff3-460e-9f4d-ae713b6daeb4
2019-05-22 12:58:16.766  INFO 15308 --- [enerContainer-1] c.d.p.s.impl.ProductConsumerListener     : >>> 接收到mq消息队列，消息编号:19e7c2fa-5bf6-462f-a862-14bc5e32cd19 ,消息内容:{"buyerId":10029122,"createTime":1558501096714,"list":[{"id":101,"num":2},{"id":102,"num":2}],"messageId":"19e7c2fa-5bf6-462f-a862-14bc5e32cd19","orderCode":"07e87fcd-16ca-49dc-b31f-1dbeb8878da0","orderPrice":18026.00}

如order工程的日志：
2019-05-22 13:00:25.684  INFO 15660 --- [nio-8082-exec-4] c.d.order.service.impl.OrderServiceImpl  : >>> 预发送消息,消息编号:30c6a800-5f8a-411d-b85b-f919e4980aff
2019-05-22 13:00:25.686  INFO 15660 --- [nio-8082-exec-4] c.d.order.service.impl.OrderServiceImpl  : >>> 插入订单,订单编号:9
2019-05-22 13:00:25.723  INFO 15660 --- [nio-8082-exec-4] c.d.order.service.impl.OrderServiceImpl  : >>> 确认并且发送消息到实时消息中间件,消息编号:30c6a800-5f8a-411d-b85b-f919e4980aff

通过上述示例体会三个中间件的作用和使用逻辑！