title: JMS消息队列
date: 2018-01-23
tags: design
categories: java

---

JMS(Java Messaging Service)是java平台上有关面向消息中间件的技术规范，它便于消息系统中的java应用程序进行消息交换，并且通过提供标准的产生、发送、接收消息的接口简化企业应用的开发

JMS类似JDBC，sun提供接口，由各个厂商（provider）来进行实现，市面上众多成熟的JMS规范实现的框架Kafk,rabbitMQ，activeMQ，zeroMQ，rocketMQ等

JMS的队列消息（Queue）传递过程如图所示
![](\img\img\20161211174534592.png)
![](\img\img\20161211174534592.png)

**对于queue模式，一个发布者发布消息，下面的接收者按队列顺序接收，比如发布了10个消息，两个接收者A、B，那就是A、B总共会收到10条消息，不重复**

JMS的主题消息（topic）传递过程如图所示
![](\img\img\20161211174552910.png)
![](\img\img\20161211174552910.png)

**对于topic模式，一个发布者发布消息，有两个接收者A、B来订阅，那么发布了10条消息，A、B各收到10条消息**

代码示例如下

### Queue模式实践:
**消息生产者:**

	public class Sender {  
	    public static void main(String[] args) throws JMSException, InterruptedException {  
	        // ConnectionFactory ：连接工厂，JMS 用它创建连接  
	        //61616是ActiveMQ默认端口
	        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(  
	                                                  ActiveMQConnection.DEFAULT_USER,  
	                                                  ActiveMQConnection.DEFAULT_PASSWORD,  
	                                                  "tcp://localhost:61616");  
	        // Connection ：JMS 客户端到JMS Provider 的连接  
	        Connection connection =  connectionFactory.createConnection();  
	        connection.start();  
	        // Session： 一个发送或接收消息的线程  
	        Session session = connection.createSession(true, Session.AUTO_ACKNOWLEDGE);  
	        // Destination ：消息的目的地;消息发送给谁.  
	        Destination destination =  session.createQueue("my-queue");  
	        // MessageProducer：消息发送者  
	        MessageProducer producer =  session.createProducer(destination);  
	        // 设置不持久化，可以更改  
	        producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);  
	        for(int i=0;i<10;i++){  
	            //创建文本消息  
	            TextMessage message = session.createTextMessage("hello.I am producer, this is a test message"+i);  
	            Thread.sleep(1000);  
	            //发送消息  
	            producer.send(message);  
	        }  
	        session.commit();  
	        session.close();  
	        connection.close();  
	    }  
	}  

**消息接收者**

	// ConnectionFactory ：连接工厂，JMS 用它创建连接  
	   private static ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(ActiveMQConnection.DEFAULT_USER,  
	           ActiveMQConnection.DEFAULT_PASSWORD, "tcp://localhost:61616");  
	   public static void main(String[] args) throws JMSException {  
	        // Connection ：JMS 客户端到JMS Provider 的连接  
	        final Connection connection =  connectionFactory.createConnection();  
	        connection.start();  
	        // Session： 一个发送或接收消息的线程  
	        final Session session = connection.createSession(true, Session.AUTO_ACKNOWLEDGE);  
	        // Destination ：消息的目的地;消息送谁那获取.  
	        Destination destination =  session.createQueue("my-queue");  
	        // 消费者，消息接收者  
	        MessageConsumer consumer1 =  session.createConsumer(destination);  
	        consumer1.setMessageListener(new MessageListener() {  
	                @Override  
	                public void onMessage(Message msg) {  
	                    try {  
	
	                        TextMessage message = (TextMessage)msg ;  
	                        System.out.println("consumerOne收到消息： "+message.getText());  
	                        session.commit();  
	                    } catch (JMSException e) {                
	                        e.printStackTrace();  
	                    }  
	                }  
	            });  
	}  

运行之后控制台不会退出一直监听消息库，对于消息发送者的十条信息，控制输出：


> consumerOne收到消息： hello.I am producer, this is a test message0 
consumerOne收到消息： hello.I am producer, this is a test message1 
consumerOne收到消息： hello.I am producer, this is a test message2 
consumerOne收到消息： hello.I am producer, this is a test message3 
consumerOne收到消息： hello.I am producer, this is a test message4 
consumerOne收到消息： hello.I am producer, this is a test message5 
consumerOne收到消息： hello.I am producer, this is a test message6 
consumerOne收到消息： hello.I am producer, this is a test message7 
consumerOne收到消息： hello.I am producer, this is a test message8 
consumerOne收到消息： hello.I am producer, this is a test message9 

**如果此时另外一个线程也存在消费者监听该Queue，则两者交换输出，共输出10条**

### Topic模式实现
**消息发布者**

	public static void main(String[] args) throws JMSException, InterruptedException {
	        // ConnectionFactory ：连接工厂，JMS 用它创建连接  
	        //61616是ActiveMQ默认端口
	        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(  
	                                                  ActiveMQConnection.DEFAULT_USER,  
	                                                  ActiveMQConnection.DEFAULT_PASSWORD,  
	                                                  "tcp://localhost:61616");  
	        // Connection ：JMS 客户端到JMS Provider 的连接  
	        Connection connection =  connectionFactory.createConnection();  
	        connection.start();  
	        // Session： 一个发送或接收消息的线程  
	        Session session = connection.createSession(true, Session.AUTO_ACKNOWLEDGE);  
	        // Destination ：消息的目的地;消息发送给谁.  
	        //Destination destination =  session.createQueue("my-queue");  
	        Destination destination = session.createTopic("STOCKS.myTopic"); //创建topic   myTopic
	        // MessageProducer：消息发送者  
	        MessageProducer producer =  session.createProducer(destination);  
	        // 设置不持久化，可以更改  
	        producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);  
	        for(int i=0;i<10;i++){  
	            //创建文本消息  
	            TextMessage message = session.createTextMessage("hello.I am producer, this is a test message"+i);  
	            //发送消息  
	            producer.send(message);  
	        }  
	        session.commit();  
	        session.close();  
	        connection.close();
	    }  

**消息订阅者**

	private static ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(ActiveMQConnection.DEFAULT_USER,
	            ActiveMQConnection.DEFAULT_PASSWORD, "tcp://localhost:61616");
	    public void run() {
	        // Connection ：JMS 客户端到JMS Provider 的连接
	        try {
	            final Connection connection = connectionFactory.createConnection();
	            connection.start();
	            // Session： 一个发送或接收消息的线程
	            final Session session = connection.createSession(true, Session.AUTO_ACKNOWLEDGE);
	            // Destination ：消息的目的地;消息送谁那获取.
	            // Destination destination = session.createQueue("my-queue");
	            Destination destination = session.createTopic("STOCKS.myTopic"); // 创建topic// myTopic
	            // 消费者，消息接收者
	            MessageConsumer consumer1 = session.createConsumer(destination);
	            consumer1.setMessageListener(new MessageListener() {
	                public void onMessage(Message msg) {
	                    try {
	                        TextMessage message = (TextMessage) msg;
	                        System.out.println("consumerOne收到消息： " + message.getText());
	                        session.commit();
	                    } catch (JMSException e) {
	                        e.printStackTrace();
	                    }
	                }
	            });
	            // 再来一个消费者，消息接收者
	            MessageConsumer consumer2 = session.createConsumer(destination);
	            consumer2.setMessageListener(new MessageListener() {
	                public void onMessage(Message msg) {
	                    try {
	                        TextMessage message = (TextMessage) msg;
	                        System.out.println("consumerTwo收到消息： " + message.getText());
	                        session.commit();
	                    } catch (JMSException e) {
	                        e.printStackTrace();
	                    }
	                }
	            });
	        } catch (Exception e) {
	        }
	    }

最后消息会重复输出: 

> consumerOne收到消息： hello.I am producer, this is a test message0 
consumerTwo收到消息： hello.I am producer, this is a test message0 
consumerOne收到消息： hello.I am producer, this is a test message1 
consumerTwo收到消息： hello.I am producer, this is a test message1 
consumerOne收到消息： hello.I am producer, this is a test message2 
consumerTwo收到消息： hello.I am producer, this is a test message2 
consumerOne收到消息： hello.I am producer, this is a test message3 
consumerTwo收到消息： hello.I am producer, this is a test message3 
consumerOne收到消息： hello.I am producer, this is a test message4 
consumerTwo收到消息： hello.I am producer, this is a test message4 
consumerOne收到消息： hello.I am producer, this is a test message5 
consumerTwo收到消息： hello.I am producer, this is a test message5 
consumerOne收到消息： hello.I am producer, this is a test message6 
consumerTwo收到消息： hello.I am producer, this is a test message6 
consumerOne收到消息： hello.I am producer, this is a test message7 
consumerTwo收到消息： hello.I am producer, this is a test message7 
consumerOne收到消息： hello.I am producer, this is a test message8 
consumerTwo收到消息： hello.I am producer, this is a test message8 
consumerOne收到消息： hello.I am producer, this is a test message9
consumerTwo收到消息： hello.I am producer, this is a test message9

我们简单总结一下使用MQ的过程:


> 1.创建与MQ的链接
2.创建消息的目的地或者来原地即Destination
3.发送消息或者制定对应的MessageListener