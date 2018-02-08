title: Redis并发锁
date: 2018-01-23
tags: design
categories: java

---
# Redis修改数据多线程并发—Redis并发锁

其实说多线程修改数据也不合适，毕竟redis服务端是单线程的，所有命令串行执行，只是在客户端并发发送命令的时候，导致串行的命令一些排列问题和网络时间差等造成数据不一致。

先配上一个简易的RedisHelper，一个set值，一个get值，一个设置并发锁，以便在我后面的操作中，你能清楚我究竟做了什么。


	public class RedisHelper {
	        public RedisClient client = new RedisClient("127.0.0.1", 6379);
	        public void Set<T>(string key, T val){
	            client.Set(key, val);
	        }
	        public T Get<T>(string key){
	            var result = client.Get<T>(key);
	            return result;
	        }
	        public IDisposable Acquire(string key){
	           return  client.AcquireLock(key);
	        }
	    }

下面看一下并发代码，我只new了两个Thread。两个线程同时想访问同一个key,分别访问五万次，在并发条件下，我们很难保证数据的准确性，请比较输出结果。

	static void Main(string[] args) {
	            RedisHelper rds = new RedisHelper();
	            rds.Set<int>("mykey1", 0);
	            Thread myThread1 = new Thread(AddVal);
	            Thread myThread2 = new Thread(AddVal);
	            myThread1.Start();
	            myThread2.Start();
	            Console.WriteLine("等待两个线程结束");
	            Console.ReadKey();
	        }
	        public static void AddVal() {
	            RedisHelper rds = new RedisHelper();
	            for (int i = 0; i < 50000; i++){
	                    int result = rds.Get<int>("mykey1");
	                    rds.Set<int>("mykey1", result + 1);
	            }
	            Console.WriteLine("线程结束，输出" + rds.Get<int>("mykey1"));
	        }

![](\img\redis_1.png)
![](../img/redis_1.png)
是的，和我们单线程，跑两个50000,会输出100000。现在是两个并发线程同时跑在由于并发造成的数据结果往往不是我们想要的。那么如何解决这个问题呢，Redis已经为我们准备好了！

你可以看到我RedisHelper中有个方法是 public IDisposable Acquire(string key)。  也可以看到他返回的是IDisposable，证明我们需要手动释放资源。方法内部的 AcquireLock正是关键之处，它向redis中索取一把锁头，**被锁住的资源，只能被单个线程访问，不会被两个线程同时get或者set,这两个线程一定是交替着进行的，当然这里的交替并不是指你一次我一次，也可能是你多次，我一次**，下面看代码。

	static void Main(string[] args) {
	            RedisHelper rds = new RedisHelper();
	            rds.Set<int>("mykey1", 0);
	            Thread myThread1 = new Thread(AddVal);
	            Thread myThread2 = new Thread(AddVal);
	            myThread1.Start();
	            myThread2.Start();
	            Console.WriteLine("等待两个线程结束");
	            Console.ReadKey();
	        }
	        public static void AddVal() {
	            RedisHelper rds = new RedisHelper();
	            for (int i = 0; i < 50000; i++) {
	                using (rds.Acquire("lock")) {
	                    int result = rds.Get<int>("mykey1");
	                    rds.Set<int>("mykey1", result + 1);
	                }
	            }
	            Console.WriteLine("线程结束，输出" + rds.Get<int>("mykey1"));
	        }

可以看到我使用了using,调用我的Acquire方法获取锁。
![](\img\redis_2.png)
![](../img/redis_2.png)
输出结果最后是100000，正是我们要的正确结果。前面的8W+是因为两个线程之一先执行结束了。

还有，在正式使用的过程中，建议给我们的锁，使用后删除掉，并加上一个过期时间，使用expire。

以免程序执行期间意外退出，导致锁一直存在，今后可能无法更新或者获取此被锁住的数据。

你也可以尝试一下不设置expire，在程序刚开始执行时，关闭console,重新运行程序，并且在redis-cli的操作控制台，get你锁住的值，将会永远获取不到。

分布式锁，慎用：所有连接此redis实例的机器，同一时刻，只能有一个获取指定name的锁.

下面是StackExchange.Redis的写法

	var token = "name-"+Environment.MachineName;
	            //如果5秒不释放锁 自动释放。避免死锁
	            if (db.LockTake("name", token, TimeSpan.FromSeconds(5))) {
	                try {
	                    //do something
	                }
	                catch (Exception ex) {
	                    //log ex
	                    //throw;
	                } finally {
	                    //主动释放
	                    db.LockRelease("name", token);
	                }
	            }

