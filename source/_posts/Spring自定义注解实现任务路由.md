title: Spring自定义注解实现任务路由
date: 2018-01-25
categories: Spring Boot

---
在Spring mvc的开发中，我们可以通过RequestMapping来配，当前方法用于处理哪一个URL的请求.同样我们现在有一个需求，有一个任务调度器，可以按照不同的任务类型路由到不同的任务执行器。其本质就是通过外部参数进行一次路由和Spring mvc做的事情类似。简单看了Spring mvc的实现原理之后，决定使用自定义注解的方式来实现以上功能。

**自定义TaskHandler注解**

	@Target({ElementType.TYPE})
	@Retention(RetentionPolicy.RUNTIME)
	@Documented
	@Component
	public @interface TaskHandler {
	  String taskType() default "";
	}

以上定义了任务处理器的注解，其中@Component表示在spring 启动过程中，会扫描到并且注入到容器中。taskType表示类型。

**任务处理器定义**

	public abstract class AbstractTaskHandler {
	  /**
	   * 任务执行器
	   * @param task 任务
	   * @return 执行结果
	   */
	   public abstract BaseResult execute(Task task);
	}

以上定义了一个任务执行的处理器，其他所有的具体的任务执行器继承实现这个方法。其中Task表示任务的定义，包括任务Id，执行任务需要的参数等。

**任务处理器实现**

接下来，我们可以实现一个具体的任务处理器。

	@TaskHandler(taskType = "UserNameChanged")
	public class UserNameChangedSender extends AbstractTaskHandler {
	  @Override
	  public BaseResult execute(Task task) {
	   return new BaseResult();
	  }
	}

以上我们就实现一个用户名修改通知的任务处理器，具体的业务逻辑这里没有实现。

其中：@TaskHandler(taskType = "UserNameChanged")，这里我们指定这个Handler用于处理用户名变更的任务

**任务处理Handler注册**

	public class TaskHandlerRegister extends ApplicationObjectSupport {
	  private final static Map<String, AbstractTaskHandler> TASK_HANDLERS_MAP = new HashMap<>();
	  private static final Logger LOGGER = LoggerFactory.getLogger(TaskHandlerRegister.class);
	  @Override
	  protected void initApplicationContext(ApplicationContext context) throws BeansException {
	    super.initApplicationContext(context);
	    Map<String, Object> taskBeanMap = context.getBeansWithAnnotation(TaskHandler.class);
	    taskBeanMap.keySet().forEach(beanName -> {
	      Object bean = taskBeanMap.get(beanName);
	      Class clazz = bean.getClass();
	      if (bean instanceof AbstractTaskHandler && clazz.getAnnotation(TaskHandler.class) != null) {
	        TaskHandler taskHandler = (TaskHandler) clazz.getAnnotation(TaskHandler.class);
	        String taskType = taskHandler.taskType();
	        if (TASK_HANDLERS_MAP.keySet().contains(taskType)) {
	          throw new RuntimeException("TaskType has Exits. TaskType=" + taskType);
	        }
	        TASK_HANDLERS_MAP.put(taskHandler.taskType(), (AbstractTaskHandler) taskBeanMap.get(beanName));
	        LOGGER.info("Task Handler Register. taskType={},beanName={}", taskHandler.taskType(), beanName);
	      }
	    });
	  }
	 
	  public static AbstractTaskHandler getTaskHandler(String taskType) {
	    return TASK_HANDLERS_MAP.get(taskType);
	  }
	}

这里继承了Spring的ApplicationObjectSupport类，具体的注册过程如下

> Spring完成bean的初始化
查找spring的容器中，所有带有TaskHandler注解的bean
校验bean是否为AbstractTaskHandler类型，获取到taskType
把该bean放到TASK_HANDLERS_MAP容器中，即注册完成

**任务执行**

接下来我们来看下任务执行

	public class TaskExecutor implements Job {
	  private static final String TASK_TYPE = "taskType";
	  @Override
	  public BaseResult execute(Task task){
	    String taskType=task.getTaskType();
	    if (TaskHandlerRegister.getTaskHandler(taskType) == null) {
	      throw new RuntimeException("can't find taskHandler,taskType=" + taskType);
	    }
	    AbstractTaskHandler abstractHandler = TaskHandlerRegister.getTaskHandler(taskType);
	    return abstractHandler.execute(task);
	  }
	}

这里发起任务执行的是一个Job，具体过程如下

> 
校验该任务类型，有没有在注册中心注册相关Handler
从任务注册中心获取到对应的处理的Handelr
执行该Handelr

以上过程就完成了，可以实现基于注解的一个任务路由过程。其实现思路来自于Spring mvc的RequestMapping的设计思路