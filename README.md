
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
				"createTime": "2018-03-11 12:00:00",
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
				"createTime": "2018-03-11 12:00:00",
				"remark": "高大上的手机"
			}
		}
	]
}

