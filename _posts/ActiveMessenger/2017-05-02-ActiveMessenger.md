---
layout: post
title: ActiveMQ简介
categories: ActiveMessenger
description: 消息中间件
keywords: java, 消息中间件，ActiveMQ
---

# ActiveMQ简介 #

ActiveMQ简介是Apache出品，最流行，能力强劲的开源消息总线。
ActiveMQ在业界应用广泛，当然想要有更强大的性能和海量数据处理能力，ActiveMQ还需要不断的升级版本，80%以上的业务ActiveMQ足够满足需求，之后学习的RocketMQ更为强大，可以说ActiveMQ是核心是基础，所以我们必须掌握。


# ActiveMQ的使用 #

官网下载：[http://activemq.apache.org/](http://activemq.apache.org/),下载最新的apache-activemq-5.11.1-bin.zip

解压下载好的压缩包，需要了解activemq.xml(ActiveMQ的配置)。jetty.xml和jetty-realm.properties（配置监控ActiveMQ的控制台）


# ActiveMQ Hello World #

Sender/Receiver：

    1. 创建ConnectionFactory工厂对象，需要填入用户名，密码，以及要连接的地址，均使用默认即可，默认端口为"tcp://localhost:61616"
    2. 通过ConnectionFactory工厂对象创建一个Connection连接，并且调用Connection的start方法开启连接，Connection默认是关闭的
    3. 通过Connection对象创建Session会话（上下文环境对象），用于接收消息，参数配置1位是否启用事务，参数配置2为签收模式，一般我们设置自动签收。
    4. 通过Session创建Destination对象，指的是一个客户端用来指定生产消息目标和消费消息来源的对象，在PTP模式中Destination被称为Queue即队列；在Pub/Sub模式中，Destination被称为Topic即主题，在程序中可以使用多个Queue和Topic。
    5. 通过Session创建消息的发送和接收对象（生产者和消费者）MessageProducer/MessageConsumer。
    6. 可以使用MessageProducer的setDeliveryMode方法为其设置持久化特性和非持久化特性
    7. 最后使用JMS规范的TextMessage形式创建数据（通过Session），并用MessageProducer的send方法发送数据，客户端使用receive方法接收数据，最后关闭Connection连接

Producer（生产者）

    public class Sender {
	
	 public static void main(String[] args) {
		try {
			//第一步：建立ConnectionFactory工厂对象，需要填入用户名、密码、以及要连接的地址，均使用默认即可，默认端口为"tcp://localhost:61616"
			ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
					ActiveMQConnectionFactory.DEFAULT_USER, 
					ActiveMQConnectionFactory.DEFAULT_PASSWORD, 
					"failover:(tcp://192.168.1.111:51511,tcp://192.168.1.111:51512,tcp://192.168.1.111:51513)?Randomize=false");
			
			//第二步：通过ConnectionFactory工厂对象我们创建一个Connection连接，并且调用Connection的start方法开启连接，Connection默认是关闭的。
			Connection connection = connectionFactory.createConnection();
			connection.start();
			
			//第三步：通过Connection对象创建Session会话（上下文环境对象），用于接收消息，参数配置1为是否启用是事务，参数配置2为签收模式，一般我们设置自动签收。
			Session session = connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE);
			
			//第四步：通过Session创建Destination对象，指的是一个客户端用来指定生产消息目标和消费消息来源的对象，在PTP模式中，Destination被称作Queue即队列；在Pub/Sub模式，Destination被称作Topic即主题。在程序中可以使用多个Queue和Topic。
			Destination destination = session.createQueue("first");
			
			//第五步：我们需要通过Session对象创建消息的发送和接收对象（生产者和消费者）MessageProducer/MessageConsumer。
			MessageProducer producer = session.createProducer(null);
			
			//第六步：我们可以使用MessageProducer的setDeliveryMode方法为其设置持久化特性和非持久化特性（DeliveryMode），我们稍后详细介绍。
			//producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);
			
			//第七步：最后我们使用JMS规范的TextMessage形式创建数据（通过Session对象），并用MessageProducer的send方法发送数据。同理客户端使用receive方法进行接收数据。最后不要忘记关闭Connection连接。
			
			for(int i = 0 ; i < 500000 ; i ++){
				TextMessage msg = session.createTextMessage("我是消息内容" + i);
				// 第一个参数目标地址
				// 第二个参数 具体的数据信息
				// 第三个参数 传送数据的模式
				// 第四个参数 优先级
				// 第五个参数 消息的过期时间
				producer.send(destination, msg, DeliveryMode.NON_PERSISTENT, 0 , 1000L);
				System.out.println("发送消息：" + msg.getText());
				Thread.sleep(1000);
				
			}

			if(connection != null){
				connection.close();
			}			
		} catch (Exception e) {
			e.printStackTrace();
		}
		
	  }
    }


Consumer（消费者）


    public class Receiver {

	  public static void main(String[] args)  {
		try {
			//第一步：建立ConnectionFactory工厂对象，需要填入用户名、密码、以及要连接的地址，均使用默认即可，默认端口为"tcp://localhost:61616"
			ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
					ActiveMQConnectionFactory.DEFAULT_USER, 
					ActiveMQConnectionFactory.DEFAULT_PASSWORD, 
					"failover:(tcp://192.168.1.111:51511,tcp://192.168.1.111:51512,tcp://192.168.1.111:51513)?Randomize=false");
			
			//第二步：通过ConnectionFactory工厂对象我们创建一个Connection连接，并且调用Connection的start方法开启连接，Connection默认是关闭的。
			Connection connection = connectionFactory.createConnection();
			connection.start();
			
			//第三步：通过Connection对象创建Session会话（上下文环境对象），用于接收消息，参数配置1为是否启用是事务，参数配置2为签收模式，一般我们设置自动签收。
			Session session = connection.createSession(Boolean.FALSE, Session.AUTO_ACKNOWLEDGE);
			
			//第四步：通过Session创建Destination对象，指的是一个客户端用来指定生产消息目标和消费消息来源的对象，在PTP模式中，Destination被称作Queue即队列；在Pub/Sub模式，Destination被称作Topic即主题。在程序中可以使用多个Queue和Topic。
			Destination destination = session.createQueue("first");
			//第五步：通过Session创建MessageConsumer
			MessageConsumer consumer = session.createConsumer(destination);
			
			while(true){
				TextMessage msg = (TextMessage)consumer.receive();
				if(msg == null) break;
				System.out.println("收到的内容：" + msg.getText());
			}			
		} catch (Exception e) {
			e.printStackTrace();
		}

		
		
		
	   }
     }


# ActiveMQ安全机制 #


ActiveMQ的web管理界面：http：//127.0.0.1:8161/admin
   ActiveMQ管控台使用jetty部署，所以需要修改密码则需要到相应的配置文件（jetty-realm.properties）
ActiveMQ应该设置有安全机制，只有符合的用户才能进行发送和获取消息，所以我们也可以在activemq.xml里面添加验证配置


# Connection方法的使用 #

    Connection应该是进行客户端身份验证的地方
    当一个Connection被创建时，它的传输默认是关闭的，必须使用start方法开启。一个Connection可以建立一个或者多个session。
    当一个程序执行完成后，必须关闭之前创建的Connection，否则ActiveMQ不能释放资源，关闭一个Connection同样也关闭了session，MessageProducer和MessageConsumer。
    
    
    Connection createConnection（）；
    Connection createConnection（String userName,String password，String url）；

# Session方法使用 #

    Connection中创建一个或者多个session。session是一个发送或者接收消息的线程，可以使用session创建MessageProducer和MessageConsumer,Message。
    Session可以被事务化，也可以不被事务化，通常，可以通过向Connection上的适当创建方法传递一个布尔参数对此进行设置
    
    Session createSession（boolean transacted，int ackknowledgeMode）；
    其中transacted为使用事务标识，ackknowledgeMode为签收模式
    
    结束事务有两种方式：提交或者回滚。当一个事务提交，消息被处理。如果事务中一个步骤失败，事务就回滚，这个事务中的已经执行的动作将被撤销。在发送消息最后也必须要使用session.commit（）方法表示提交事务。
    
    签收模式有三种：
    Session.AUTO_ACKNOWLEDGE:当客户端从receive或onMessage成功返回时，session自动签收客户端的这条消息的收条
    Session.CLIENT_ACKNOWLEDGE：客户端通过调用消息的acknowledge方法签收消息，在这种情况下，签收发送在session层面：签收一个已经消费的消息会自动签收这个session所有已消费消息的收条
    Session.DUPS_ACKNOWLEDGE：指示session不必确保对传送消息的签收，可能引起消息的重复，但降低了session的开心，所以只有客户端容忍重复消息，才使用


# MessageProducer方法使用 #

    MessageProducer是一个由session创建的对象，用来向destination发送消息。
    
    void send(Destination destination,Message message);
    
    void send(Destination destination,Message message,int deliveryMode,int priority,long timeToLive);
    
    void send(Message message);
    
    void send(Message message,int deliveryMode,int priority,long timeToLive);
    
    deliveryMode传送模式：PERSISTENT和NON_PERSISTENT两种；默认是持久性消息。如果容忍消息丢失，那么使用非持久性消息可以减少存储开销和改善性能
    priority消息优先级：从0-9十个级别，0-4是普通消息，5-9是加急消息，默认是4，JMS不要求严格按照这十个优先级发送消息，但必须保证加急消息要优先于普通消息到达
    timeToLive消息过期时间：默认永不过期。如果消息在特点周期失去意义，那么可以设置过期时间，单位为毫秒。


# MessageConsumer方法使用 #

MessageConsumer是一个由session创建的对象，用来接收destination的消息

    MessageConsumer createConsumer（Destination destination）
    MessageConsumer createConsumer（Destination destination，String messageSelector）；
    MessageConsumer createConsumer（Destination destination，String messageSelector，boolean noLocal）；
    MessageConsumer createDurableSubscriber（Topic topic，String name）；
    MessageConsumer createDurableSubscriber（Topic topic，String name，String messageSelector，boolean noLocal)；
    
    messageSelector消息选择器：noLocal标志默认为false，当设置为true时限制消费者只能接收和自己相同的连接（Connection）所发布的消息，此标志只适应于主题，不适用队列：name标识订阅主题所对应的订阅名称，持久订阅时需要设置此参数。
    
    public final String SELECTOR="JMS_TYPE='MY_TAG1'";该选择器检查了传入消息的JMS_TYPE属性，并确定这个属性是否等于MY_TAG1。如果相等，则消息被消费；如果不等，那么消息被忽略。


消息的同步和异步接收：

> 消息的同步接收是指客户端主动去接收消息，客户端可以采用MessageConsumer的receive方法去接收下一个消息。
> Message receive（）
> Message receive（long timeout）
> Message receiveNoWait（）
> 
> 消息的异步接收是指当消息到达时，ActiveMQ主动通知客户端，可以注册一个实现MessageListener接口的对象到MessageConsumer，MessageListener只有一个必须实现的方法-onMessage，它只接收一个参数即message。在为每个发送到Destination的消息实现onMessage，将调用该方法

----------

    public class AppConsumer {
    public static final String url = "tcp://192.168.1.2:61616";
    public static final String queueName = "queue-test";
    public static void main(String[] args) throws JMSException {
        //创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(url);
        //创建连接
        Connection connection = connectionFactory.createConnection();
        //启动连接
        connection.start();
        //创建会话(单一线程)
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        //创建目标
        Destination destination = session.createQueue(queueName);
        //创建消费者
        MessageConsumer consumer = session.createConsumer(destination);
        //创建一个监听器
        consumer.setMessageListener(new MessageListener() {
            public void onMessage(Message message) {
                try {
                    TextMessage textMessage = (TextMessage) message;
                    System.out.println(textMessage.getText()+"i");
                } catch (JMSException e) {
                    e.printStackTrace();
                }
            }
        });
     }
    }


# Message #

    Message由：消息头，属性，消息体
    
    
    BlobMessage createBlobMessage(File file)
    BlobMessage createBlobMessage(InputStream in)
    BlobMessage createBlobMessage(URL url)
    BlobMessage createBlobMessage(URL url,boolean deletedByBroker)
    BytesMessage createBytesMessage()
    MapMessage createMapMessage()
    Message createMessage()
    ObjectMessage createObjectMessage()
    ObjectMessage createObjectMessage(Serializable object)
    TextMessage createTextMessage()
    TextMessage createTextMessage(String text)
    
    我们一般会在接收端通过instanceof方法去区别数据类型。

# 创建临时消息 #

    ActiveMQ通过createTemporaryQueue()和createTemporaryTopic()创建临时目标，这些目标持续到connection关闭。只有创建临时目标的connection所创建的客户端才可以从临时目标接收消息，但是任何的生产者都可以向临时目标中发送消息。如果关闭了创建此目标的connection，那么临时目标被关闭，内容也将消失。
    TemporaryQueue createTemporaryQueue()
    TemporaryTopic createTemporaryTopic()


# 高级主题（点对点） #

queue就是一个点对点模型，即一条消息只被一个消费者消费

# 高级主题（Pub/Sub） #

一条消息被多个消费者消费

# 高级主题（与spring进行整合） #

使用spring框架整合ActiveMQ，比如异步消费数据，异步发送邮件，异步做查询操作等。。。

https://blog.csdn.net/jiangxuchen/article/details/8004570







