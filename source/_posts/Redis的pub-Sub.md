title: Redis的pub/Sub
date: 2018-01-24 
tags: design
categories: java

---
## 1.什么是pub/sub

Pub/Sub功能（means Publish, Subscribe）即发布及订阅功能。
基于事件的系统中，Pub/Sub是目前广泛使用的通信模型，它采用事件作为基本的通信机制，提供大规模系统所要求的**松散耦合的交互模式**：
订阅者(如客户端)以事件订阅的方式表达出它有兴趣接收的一个事件或一类事件；
发布者(如服务器)可将订阅者感兴趣的事件随时通知相关订阅者。
熟悉设计模式的朋友应该了解这与23种设计模式中的观察者模式极为相似。 
同样,Redis的pub/sub是一种消息通信模式，**主要的目的是解除消息发布者和消息订阅者之间的耦合**,
Redis作为一个pub/sub的server,**在订阅者和发布者之间起到了消息路由的功能**。

## 2.Redis pub/sub的实现
Redis通过publish和subscribe命令实现订阅和发布的功能。
订阅者可以通过subscribe向redis server订阅自己感兴趣的消息类型。redis将信息类型称为**通道(channel)。**
当发布者通过publish命令向redis server发送特定类型的信息时，订阅该消息类型的全部订阅者都会收到此消息。

客户端1订阅CCTV1:

	127.0.0.1:6379> subscribe CCTV1
	Reading messages... (press Ctrl-C to quit)
	1) "subscribe"
	2) "CCTV1"
	3) (integer) 1

客户端2订阅CCTV1和CCTV2:

	127.0.0.1:6379> subscribe CCTV1 CCTV2
	Reading messages... (press Ctrl-C to quit)
	1) "subscribe"
	2) "CCTV1"
	3) (integer) 1
	1) "subscribe"
	2) "CCTV2"
	3) (integer) 2

此时这两个客户端分别监听这指定的频道。现在另一个客户端向服务器推送了关于这两个频道的信息。

	127.0.0.1:6379> publish CCTV1 "cctv1 is good"
	(integer) 2
	//返回2表示两个客户端接收了次消息。被接收到消息的客户端如下所示。
	1) "message"
	2) "CCTV1"
	3) "cctv1 is good"
	----
	1) "message"
	2) "CCTV1"
	3) "cctv1 is good"

如上的订阅/发布也称订阅发布到频道(**使用publish与subscribe命令**)，此外还有订阅发布到模式(**使用psubscribe来订阅一个模式**)

	127.0.0.1:6379> psubscribe CCTV*
	Reading messages... (press Ctrl-C to quit)
	1) "psubscribe"
	2) "CCTV*"
	3) (integer) 1

当依然先如上推送一个CCTV1的消息时，该客户端正常接收。

**Pub/Sub在java中的实现:**

	dependencies {
	    compile 'redis.clients:jedis:2.4.2'
	}

Redis驱动包提供了一个抽象类:JedisPubSub…继承这个类就完成了对客户端对订阅的监听。示例代码:

	public class TestPubSub extends JedisPubSub {
	    @Override
	    public void onMessage(String channel, String message) {
	        // TODO Auto-generated method stub
	        System.out.println(channel + "," + message);
	    }
	    @Override
	    public void onPMessage(String pattern, String channel, String message) {
	        // TODO Auto-generated method stub
	        System.out.println(pattern + "," + channel + "," + message);
	    }
	    @Override
	    public void onSubscribe(String channel, int subscribedChannels) {
	        // TODO Auto-generated method stub
	        System.out.println("onSubscribe: channel[" + channel + "]," + "subscribedChannels[" + subscribedChannels + "]");
	    }
	    @Override
	    public void onUnsubscribe(String channel, int subscribedChannels) {
	        // TODO Auto-generated method stub
	        System.out.println(
	                "onUnsubscribe: channel[" + channel + "], " + "subscribedChannels[" + subscribedChannels + "]");
	    }
	    @Override
	    public void onPUnsubscribe(String pattern, int subscribedChannels) {
	        // TODO Auto-generated method stub
	        System.out.println("onPUnsubscribe: pattern[" + pattern + "]," +
	                "subscribedChannels[" + subscribedChannels + "]");
	    }
	    @Override
	    public void onPSubscribe(String pattern, int subscribedChannels) {
	        System.out.println("onPSubscribe: pattern[" + pattern + "], " +
	                "subscribedChannels[" + subscribedChannels + "]");
	    }
	}

如上所示,抽象类中存在六个方法。分别表示


> 监听到**订阅模式**接受到消息时的回调 (onPMessage)
> 监听到**订阅频道**接受到消息时的回调 (onMessage )
> **订阅频道时**的回调( onSubscribe )
> **取消订阅频道时**的回调( onUnsubscribe )
> **订阅模式时**的回调 ( onPSubscribe )
> **取消订阅模式时**的回调( onPUnsubscribe )

运行我们刚刚编写的类:

	@Test
    public void pubsubjava() {
        // TODO Auto-generated method stub
        Jedis jr = null;
        try {       
        	jr = new Jedis("127.0.0.1", 6379, 0);// redis服务地址和端口号
            jr.auth("wx950709");
            TestPubSub sp = new TestPubSub();
            // jr客户端配置监听两个channel
            sp.subscribe(jr.getClient(), "news.share", "news.blog");
        } catch (Exception e) {
            e.printStackTrace();
        } finally{
            if(jr!=null){
                jr.disconnect();
            }
        }
    }

从代码中我们不难看出，我们声明的一个redis链接在设置监听后就可以执行一些操作，例如发布消息，订阅消息等。。。 
当运行上述代码后会在控制台输出:

> onSubscribe: channel[news.share],subscribedChannels[1]
> onSubscribe: channel[news.blog],subscribedChannels[2]
> //onSubscribe方法成功运行

当在此时有客户端向new.share或者new.blog通道publish消息时，onMessage方法即可被响应。(**jedis.publish(channel, message)**)。

**Pub/Sub在Spring中的实践 **
导入依赖jar

	dependencies {
	    compile 'org.springframework.data:spring-data-redis:1.7.2.RELEASE'
	    compile 'redis.clients:jedis:2.4.2'
	}

Spring配置redis的链接:

	@Configuration
	public class AppConfig {
	    @Bean
	    JedisConnectionFactory jedisConnectionFactory() {
	        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory();
	        jedisConnectionFactory.setPassword("xxxx");
	        return jedisConnectionFactory;
	    }
	    @Bean
	    RedisTemplate<String, Object> redisTemplate() {
	        final RedisTemplate<String, Object> template = new RedisTemplate<String, Object>();
	        template.setConnectionFactory(jedisConnectionFactory());
	        template.setDefaultSerializer(new StringRedisSerializer());
	        return template;
	    }
	    @Bean
	    MessageListenerAdapter messageListener() {
	        return new MessageListenerAdapter(new RedisMessageListener());
	    }
	    @Bean
	    RedisMessageListenerContainer redisContainer() {
	        final RedisMessageListenerContainer container = new RedisMessageListenerContainer();
	        container.setConnectionFactory(jedisConnectionFactory());
	        container.addMessageListener(messageListener(), topic());
	        return container;
	    }
	    @Bean
	    RedisPublisherImpl redisPublisher() {
	        return new RedisPublisherImpl(redisTemplate(), topic());
	    }
	    @Bean
	    ChannelTopic topic() {
	        return new ChannelTopic( "pubsub:queue" );
	    }
	}

如上的配置即配置了对Redis的链接。在RedisTemplate中没有设置ip端口等信息则全部为默认的。**在配置类中的将ChannelTopic加入IOC容器**。则在Spring启动时会在一个RedisTemplate(一个对Redis的链接)中设置的一个channel，即**pubsub:queue**。 
在上述配置中，**RedisMessageListener**是我们生成的，这个类即为核心监听类，RedisTemplate接受到数据如何处理就是在该类中处理的。

	public class RedisMessageListener implements MessageListener {
	        @Override
	        public void onMessage( final Message message, final byte[] pattern ) {
	            System.out.println("Message received: " + message.toString() );
	        }
	}

现在我们在获取RedisTemplate,并给pubsub:queue这个channel publish数据。

	public class PubSubMain {
	    RedisTemplate<String,Object> redisTemplate;
	    public  void execute() {
	       String channel = "pubsub:queue";
	      redisTemplate.convertAndSend(channel, "from testData");
	    }
	    public static void main(String[] args) {
	        ApplicationContext applicationContext   = new AnnotationConfigApplicationContext(AppConfig.class);
	        PubSubMain pubSubMain = new PubSubMain();
	        pubSubMain.redisTemplate = (RedisTemplate<String, Object>) applicationContext.getBean("redisTemplate");
	        pubSubMain.execute();
	    }
	}

此时运行main 方法:

	Message received: from app 12
	//表明接受成功，当在命令行中启动一个客户端并publish时依然可以在客户端打印出message