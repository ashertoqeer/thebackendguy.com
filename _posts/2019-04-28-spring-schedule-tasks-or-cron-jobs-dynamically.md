---
layout: post
title: "Java Spring Schedule Tasks or Cron Jobs Dynamically"
permalink: "spring-schedule-tasks-or-cron-jobs-dynamically/"
last_modified_at: 2019-04-28T00:00:00
excerpt: "Spring provides excellent support for scheduling tasks. You can simple schedule tasks or use cron expressions, either at compile time or dynamically at run time."
category: "java-spring-framework"
---
Spring provides **Task Scheduler** API for scheduling tasks or cron jobs dynamically. It could be directly injected to any bean given that you have **@EnableScheduling** in your configuration. It takes a **Runnable** to execute in future. It provides different methods to schedule task.

```java
public interface TaskScheduler {

	// Run once at given timestamp
        ScheduledFuture schedule(Runnable task, Date startTime);

	// First run at given timestamp, then repeatedly after period 
        ScheduledFuture scheduleAtFixedRate(Runnable task, Date startTime, long period);

	// Run as soon as possible, then repeatedly after period
        ScheduledFuture scheduleAtFixedRate(Runnable task, long period);

	// First run at given timestamp,then repeatedly after delay from last execution
        ScheduledFuture scheduleWithFixedDelay(Runnable task, Date startTime, long delay);

	// Run as soon as possible, then repeatedly after delay from last execution
        ScheduledFuture scheduleWithFixedDelay(Runnable task, long delay);
        
        // Run as per trigger conditions
        ScheduledFuture schedule(Runnable task, Trigger trigger);

}
```



**Trigger** provides much more flexibility. **Spring** has two implementations for **trigger**. 

1. CronTrigger
2. PeriodicTrigger

**CronTrigger** is used to execute task on the base of cron expression.

```
// execute task every day at 8:00 am
scheduler.schedule(task, new CronTrigger("0 0 8 * * ?"));
```

**PeriodicTrigger** is used to schedule periodic task. 

```
// execute task every 10 days
scheduler.schedule(task, new PeriodicTrigger(10, TimeUnit.DAYS));
```

You can also specify other values like initial delay value and a boolean to indicate whether the period should be interpreted as a fixed-rate or a fixed-delay.

**Note: ** All schedule jobs would have to be rescheduled in case of context refresh, i.e server restart. You can listen to context refresh event to reschedule jobs.

Here is an example service:

### Example: 

```java
import java.util.HashMap;
import java.util.Map;
import java.util.TimeZone;
import java.util.concurrent.ScheduledFuture;

import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.TaskScheduler;
import org.springframework.scheduling.support.CronTrigger;
import org.springframework.stereotype.Service;

@Service
public class ScheduleTaskService {

	// Task Scheduler
	TaskScheduler scheduler;
	
	// A map for keeping scheduled tasks
	Map<Integer, ScheduledFuture<?>> jobsMap = new HashMap<>();
	
	public ScheduleTaskService(TaskScheduler scheduler) {
		this.scheduler = scheduler;
	}
	
	
	// Schedule Task to be executed every night at 00 or 12 am
	public void addTaskToScheduler(int id, Runnable task) {
		ScheduledFuture<?> scheduledTask = scheduler.schedule(task, new CronTrigger("0 0 0 * * ?", TimeZone.getTimeZone(TimeZone.getDefault().getID())));
		jobsMap.put(id, scheduledTask);
	}
	
	// Remove scheduled task 
	public void removeTaskFromScheduler(int id) {
		ScheduledFuture<?> scheduledTask = jobsMap.get(id);
		if(scheduledTask != null) {
			scheduledTask.cancel(true);
			jobsMap.put(id, null);
		}
	}
	
	// A context refresh event listener
	@EventListener({ ContextRefreshedEvent.class })
	void contextRefreshedEvent() {
		// Get all tasks from DB and reschedule them in case of context restarted
	}
}
```
