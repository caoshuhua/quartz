# quartz
在实际项目应用中经常会用到定时任务，可以通过quartz和spring的简单配置即可完成，但如果要改变任务的执行时间、频率，废弃任务等就需要改变配置甚至代码需要重启服务器，这里介绍一下如何通过quartz与spring的组合实现动态的改变定时任务的状态的一个实现。
参考文章：http://www.meiriyouke.net/?p=82
本文章适合对quartz和spring有一定了解的读者。
spring版本为3.2  quartz版本为2.2.1  如果使用了quartz2.2.1 则spring版本需3.1以上
1.
spring中引入注册bean

1

2
<bean id="schedulerFactoryBean"
        class="org.springframework.scheduling.quartz.SchedulerFactoryBean" />
为什么要与spring结合？
与spring结合可以使用spring统一管理quartz中任务的生命周期，使得web容器关闭时所有的任务一同关闭。如果不用spring管理可能会出现web容器关闭而任务仍在继续运行的情况，不与spring结合的话要自己控制任务在容器关闭时一起关闭。
2.创建保存计划任务信息的实体类


1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41

42

43

44

45

46

47

48

49

50

51

52

53

54

55

56
/** 
 *  
* @Description: 计划任务信息 
* @author snailxr 
* @date 2014年4月24日 下午10:49:43 
 */  
public class ScheduleJob {  
   
    public static final String STATUS_RUNNING = "1";  
    public static final String STATUS_NOT_RUNNING = "0";  
    public static final String CONCURRENT_IS = "1";  
    public static final String CONCURRENT_NOT = "0";  
    private Long jobId;  
   
    private Date createTime;  
   
    private Date updateTime;  
    /** 
     * 任务名称 
     */  
    private String jobName;  
    /** 
     * 任务分组 
     */  
    private String jobGroup;  
    /** 
     * 任务状态 是否启动任务 
     */  
    private String jobStatus;  
    /** 
     * cron表达式 
     */  
    private String cronExpression;  
    /** 
     * 描述 
     */  
    private String description;  
    /** 
     * 任务执行时调用哪个类的方法 包名+类名 
     */  
    private String beanClass;  
    /** 
     * 任务是否有状态 
     */  
    private String isConcurrent;  
    /** 
     * spring bean 
     */  
    private String springId;  
    /** 
     * 任务调用的方法名 
     */  
    private String methodName;  
   
    //get  set.......  
}
该实体类与数据库中的表对应，在数据库中存储多个计划任务。
  注意：jobName 跟 groupName的组合应该是唯一的,beanClass springId至少有一个
 
在项目启动时运行以下代码：

1

2

3

4

5

6

7

8

9

10

11
public void init() throws Exception {
 
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
 
        // 这里从数据库中获取任务信息数据
        List<ScheduleJob> jobList = scheduleJobMapper.getAll();
     
        for (ScheduleJob job : jobList) {
            addJob(job);
        }
    }

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41
/**
     * 添加任务
     * 
     * @param scheduleJob
     * @throws SchedulerException
     */
    public void addJob(ScheduleJob job) throws SchedulerException {
        if (job == null || !ScheduleJob.STATUS_RUNNING.equals(job.getJobStatus())) {
            return;
        }
 
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        log.debug(scheduler + ".......................................................................................add");
        TriggerKey triggerKey = TriggerKey.triggerKey(job.getJobName(), job.getJobGroup());
 
        CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
 
        // 不存在，创建一个
        if (null == trigger) {
            Class clazz = ScheduleJob.CONCURRENT_IS.equals(job.getIsConcurrent()) ? QuartzJobFactory.class : QuartzJobFactoryDisallowConcurrentExecution.class;
 
            JobDetail jobDetail = JobBuilder.newJob(clazz).withIdentity(job.getJobName(), job.getJobGroup()).build();
 
            jobDetail.getJobDataMap().put("scheduleJob", job);
 
            CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(job.getCronExpression());
 
            trigger = TriggerBuilder.newTrigger().withIdentity(job.getJobName(), job.getJobGroup()).withSchedule(scheduleBuilder).build();
 
            scheduler.scheduleJob(jobDetail, trigger);
        } else {
            // Trigger已存在，那么更新相应的定时设置
            CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(job.getCronExpression());
 
            // 按新的cronExpression表达式重新构建trigger
            trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();
 
            // 按新的trigger重新设置job执行
            scheduler.rescheduleJob(triggerKey, trigger);
        }
    }
看到代码第20行根据scheduleJob类中CONCURRENT_IS来判断任务是否有状态。来给出不同的Job实现类


1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17
/**
 * 
 * @Description: 若一个方法一次执行不完下次轮转时则等待改方法执行完后才执行下一次操作
 * @author snailxr
 * @date 2014年4月24日 下午5:05:47
 */
@DisallowConcurrentExecution
public class QuartzJobFactoryDisallowConcurrentExecution implements Job {
    public final Logger log = Logger.getLogger(this.getClass());
 
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        ScheduleJob scheduleJob = (ScheduleJob) context.getMergedJobDataMap().get("scheduleJob");
        TaskUtils.invokMethod(scheduleJob);
 
    }
}

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15
/**
 * 
 * @Description: 计划任务执行处 无状态
 * @author snailxr
 * @date 2014年4月24日 下午5:05:47
 */
public class QuartzJobFactory implements Job {
    public final Logger log = Logger.getLogger(this.getClass());
 
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        ScheduleJob scheduleJob = (ScheduleJob) context.getMergedJobDataMap().get("scheduleJob");
        TaskUtils.invokMethod(scheduleJob);
    }
}
真正执行计划任务的代码就在TaskUtils.invokMethod(scheduleJob)里面
通过scheduleJob的beanClass或springId通过反射或spring来获得需要执行的类,通过methodName来确定执行哪个方法

1

2

3

4

5

6

7

8

9

10

11

12

13

14

15

16

17

18

19

20

21

22

23

24

25

26

27

28

29

30

31

32

33

34

35

36

37

38

39

40

41

42

43

44

45

46

47

48

49

50

51

52

53

54

55
public class TaskUtils {
    public final static Logger log = Logger.getLogger(TaskUtils.class);
 
    /**
     * 通过反射调用scheduleJob中定义的方法
     * 
     * @param scheduleJob
     */
    public static void invokMethod(ScheduleJob scheduleJob) {
        Object object = null;
        Class clazz = null;
                //springId不为空先按springId查找bean
        if (StringUtils.isNotBlank(scheduleJob.getSpringId())) {
            object = SpringUtils.getBean(scheduleJob.getSpringId());
        } else if (StringUtils.isNotBlank(scheduleJob.getBeanClass())) {
            try {
                clazz = Class.forName(scheduleJob.getBeanClass());
                object = clazz.newInstance();
            } catch (Exception e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
 
        }
        if (object == null) {
            log.error("任务名称 = [" + scheduleJob.getJobName() + "]---------------未启动成功，请检查是否配置正确！！！");
            return;
        }
        clazz = object.getClass();
        Method method = null;
        try {
            method = clazz.getDeclaredMethod(scheduleJob.getMethodName());
        } catch (NoSuchMethodException e) {
            log.error("任务名称 = [" + scheduleJob.getJobName() + "]---------------未启动成功，方法名设置错误！！！");
        } catch (SecurityException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        if (method != null) {
            try {
                method.invoke(object);
            } catch (IllegalAccessException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } catch (IllegalArgumentException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
         
    }
}
对任务的暂停，删除，修改等操作



**
     * 获取所有计划中的任务列表
     * 
     * @return
     * @throws SchedulerException
     */
    public List<ScheduleJob> getAllJob() throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        GroupMatcher<JobKey> matcher = GroupMatcher.anyJobGroup();
        Set<JobKey> jobKeys = scheduler.getJobKeys(matcher);
        List<ScheduleJob> jobList = new ArrayList<ScheduleJob>();
        for (JobKey jobKey : jobKeys) {
            List<? extends Trigger> triggers = scheduler.getTriggersOfJob(jobKey);
            for (Trigger trigger : triggers) {
                ScheduleJob job = new ScheduleJob();
                job.setJobName(jobKey.getName());
                job.setJobGroup(jobKey.getGroup());
                job.setDescription("触发器:" + trigger.getKey());
                Trigger.TriggerState triggerState = scheduler.getTriggerState(trigger.getKey());
                job.setJobStatus(triggerState.name());
                if (trigger instanceof CronTrigger) {
                    CronTrigger cronTrigger = (CronTrigger) trigger;
                    String cronExpression = cronTrigger.getCronExpression();
                    job.setCronExpression(cronExpression);
                }
                jobList.add(job);
            }
        }
        return jobList;
    }
 
    /**
     * 所有正在运行的job
     * 
     * @return
     * @throws SchedulerException
     */
    public List<ScheduleJob> getRunningJob() throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        List<JobExecutionContext> executingJobs = scheduler.getCurrentlyExecutingJobs();
        List<ScheduleJob> jobList = new ArrayList<ScheduleJob>(executingJobs.size());
        for (JobExecutionContext executingJob : executingJobs) {
            ScheduleJob job = new ScheduleJob();
            JobDetail jobDetail = executingJob.getJobDetail();
            JobKey jobKey = jobDetail.getKey();
            Trigger trigger = executingJob.getTrigger();
            job.setJobName(jobKey.getName());
            job.setJobGroup(jobKey.getGroup());
            job.setDescription("触发器:" + trigger.getKey());
            Trigger.TriggerState triggerState = scheduler.getTriggerState(trigger.getKey());
            job.setJobStatus(triggerState.name());
            if (trigger instanceof CronTrigger) {
                CronTrigger cronTrigger = (CronTrigger) trigger;
                String cronExpression = cronTrigger.getCronExpression();
                job.setCronExpression(cronExpression);
            }
            jobList.add(job);
        }
        return jobList;
    }
 
    /**
     * 暂停一个job
     * 
     * @param scheduleJob
     * @throws SchedulerException
     */
    public void pauseJob(ScheduleJob scheduleJob) throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        JobKey jobKey = JobKey.jobKey(scheduleJob.getJobName(), scheduleJob.getJobGroup());
        scheduler.pauseJob(jobKey);
    }
 
    /**
     * 恢复一个job
     * 
     * @param scheduleJob
     * @throws SchedulerException
     */
    public void resumeJob(ScheduleJob scheduleJob) throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        JobKey jobKey = JobKey.jobKey(scheduleJob.getJobName(), scheduleJob.getJobGroup());
        scheduler.resumeJob(jobKey);
    }
 
    /**
     * 删除一个job
     * 
     * @param scheduleJob
     * @throws SchedulerException
     */
    public void deleteJob(ScheduleJob scheduleJob) throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        JobKey jobKey = JobKey.jobKey(scheduleJob.getJobName(), scheduleJob.getJobGroup());
        scheduler.deleteJob(jobKey);
 
    }
 
    /**
     * 立即执行job
     * 
     * @param scheduleJob
     * @throws SchedulerException
     */
    public void runAJobNow(ScheduleJob scheduleJob) throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
        JobKey jobKey = JobKey.jobKey(scheduleJob.getJobName(), scheduleJob.getJobGroup());
        scheduler.triggerJob(jobKey);
    }
 
    /**
     * 更新job时间表达式
     * 
     * @param scheduleJob
     * @throws SchedulerException
     */
    public void updateJobCron(ScheduleJob scheduleJob) throws SchedulerException {
        Scheduler scheduler = schedulerFactoryBean.getScheduler();
 
        TriggerKey triggerKey = TriggerKey.triggerKey(scheduleJob.getJobName(), scheduleJob.getJobGroup());
 
        CronTrigger trigger = (CronTrigger) scheduler.getTrigger(triggerKey);
 
        CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(scheduleJob.getCronExpression());
 
        trigger = trigger.getTriggerBuilder().withIdentity(triggerKey).withSchedule(scheduleBuilder).build();
 
        scheduler.rescheduleJob(triggerKey, trigger);
    }
小提示
更新表达式，判断表达式是否正确可用一下代码
CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule("xxxxx");
抛出异常则表达式不正确


























或许你应该看出来了，我的项目是spring整合了mybatis，目前spring的最新版本已经到了4.x系列，但是最新版的mybatis-spring的整合插件所依赖推荐的依然是spring 3.1.3.RELEASE，所以这里没有用spring的最新版而是用了推荐的3.1.3.RELEASE，毕竟最新版本的功能一般情况下也用不到。

至于quartz，则是用了目前的最新版2.2.1

之所以在这里特别对版本作一下说明，是因为spring和quartz的整合对版本是有要求的。

spring3.1以下的版本必须使用quartz1.x系列，3.1以上的版本才支持quartz 2.x，不然会出错。

至于原因，则是spring对于quartz的支持实现，org.springframework.scheduling.quartz.CronTriggerBean继承了org.quartz.CronTrigger，在quartz1.x系列中org.quartz.CronTrigger是个类，而在quartz2.x系列中org.quartz.CronTrigger变成了接口，从而造成无法用spring的方式配置quartz的触发器（trigger）。
在Spring中使用Quartz有两种方式实现：第一种是任务类继承QuartzJobBean，第二种则是在配置文件里定义任务类和要执行的方法，类和方法可以是普通类。很显然，第二种方式远比第一种方式来的灵活。

这里采用的就是第二种方式。

spring配置文件：

<!-- 使用MethodInvokingJobDetailFactoryBean，任务类可以不实现Job接口，通过targetMethod指定调用方法-->
<bean id="taskJob" class="com.tyyd.dw.task.DataConversionTask"/>
<bean id="jobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
    <property name="group" value="job_work"/>
    <property name="name" value="job_work_name"/>
    <!--false表示等上一个任务执行完后再开启新的任务-->
    <property name="concurrent" value="false"/>
    <property name="targetObject">
        <ref bean="taskJob"/>
    </property>
    <property name="targetMethod">
        <value>run</value>
    </property>
</bean>
<!--  调度触发器 -->
<bean id="myTrigger"
      class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
    <property name="name" value="work_default_name"/>
    <property name="group" value="work_default"/>
    <property name="jobDetail">
        <ref bean="jobDetail" />
    </property>
    <property name="cronExpression">
        <value>0/5 * * * * ?</value>
    </property>
</bean>
<!-- 调度工厂 -->
<bean id="scheduler" class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
    <property name="triggers">
        <list>
            <ref bean="myTrigger"/>
        </list>
    </property>
</bean>
Task类则是一个普通的Java类，没有继承任何类和实现任何接口(当然可以用注解方式来声明bean)：

//@Component
public class DataConversionTask{
    /** 日志对象 */
    private static final Logger LOG = LoggerFactory.getLogger(DataConversionTask.class);
    public void run() {
        if (LOG.isInfoEnabled()) {
            LOG.info("数据转换任务线程开始执行");
        }
    }
}
至此，简单的整合大功告成，run方法将每隔5秒执行一次，因为配置了concurrent等于false，所以假如run方法的执行时间超过5秒，在执行完之前即使时间已经超过了5秒下一个定时计划执行任务仍不会被开启，如果是true，则不管是否执行完，时间到了都将开启。

 

接下去，将实现如何动态的修改定时执行的时间，以及如何停止正在执行的任务，待续，，，

 

顺便贴一下cronExpression表达式备忘： 字段 允许值 允许的特殊字符

秒 0-59 , – * /

分 0-59 , – * /

小时 0-23 , – * /

日期 1-31 , – * ? / L W C

月份 1-12 或者 JAN-DEC , – * /

星期 1-7 或者 SUN-SAT , – * ? / L C #

年（可选） 留空, 1970-2099 , – * /

表达式意义

"0 0 12 * * ?" 每天中午12点触发

"0 15 10 ? * *" 每天上午10:15触发

"0 15 10 * * ?" 每天上午10:15触发

"0 15 10 * * ? *" 每天上午10:15触发

"0 15 10 * * ? 2005" 2005年的每天上午10:15触发

"0 * 14 * * ?" 在每天下午2点到下午2:59期间的每1分钟触发

"0 0/5 14 * * ?" 在每天下午2点到下午2:55期间的每5分钟触发

"0 0/5 14,18 * * ?" 在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发

"0 0-5 14 * * ?" 在每天下午2点到下午2:05期间的每1分钟触发

"0 10,44 14 ? 3 WED" 每年三月的星期三的下午2:10和2:44触发

"0 15 10 ? * MON-FRI" 周一至周五的上午10:15触发

"0 15 10 15 * ?" 每月15日上午10:15触发

"0 15 10 L * ?" 每月最后一日的上午10:15触发

"0 15 10 ? * 6L" 每月的最后一个星期五上午10:15触发

"0 15 10 ? * 6L 2002-2005" 2002年至2005年的每月的最后一个星期五上午10:15触发

"0 15 10 ? * 6#3" 每月的第三个星期五上午10:15触发

每天早上6点

0 6 * * *

每两个小时

0 */2 * * *

晚上11点到早上8点之间每两个小时，早上八点

0 23-7/2，8 * * *

每个月的4号和每个礼拜的礼拜一到礼拜三的早上11点

0 11 4 * 1-3

1月1日早上4点

0 4 1 1 *
