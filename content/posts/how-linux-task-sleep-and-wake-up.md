---
title: 리눅스 태스크가 대기하고 깨어나는 방법
date: 2023-10-23 20:39:15 +0900
tags:
  - Linux
  - kernel
categories:
  - 리눅스 커널
comments: true
---
태스크는 파일 처리 등 IO 작업을 요청한 다음 대기 상태에 들어가고, 처리가 완료되면 깨어나 남은 작업을 수행한다. 너무나 당연한 과정이라 인식하고 있지도 않았지만, 리눅스 커널은 이 작업을 여러 가지 단계들로 나누어 수행하고 있다.

# 대기
대기 상태는 시그널 처리가 가능한 `TASK_INTERRUPTIBLE`와  시그널 처리가 불가능한`TASK_UNINTERRUPTIBLE` 두 가지 상태로 나누어진다. 이 상태에 들어간 태스크는 대기열에 자신의 구조체를 두고, 인터럽트가 발생하기를 기다리게 된다.

# 깨우기
인터럽트가 발생하면 커널은 대기열에 들어가 있는 모든 태스크를 깨운다. 깨어난 태스크는 발생한 이벤트와 자신이 깨어날 조건을 비교해 작업을 계속 수행할지, 아니면 대기열에 다시 들어갈지를 결정한다. 커널이 조건을 직접 확인해 스레드를 깨우는 것이 더 효율적으로 보이지만 이런 방식이 채택된 이유는 깨어날 조건이 매우 복잡할 수도 있기 때문이다.
# inotify_read
대기 및 깨우기 과정을 잘 보여주는 inotify_read 함수를 분석해 보자.

출처: [https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/fs/notify/inotify/inotify_user.c](https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/fs/notify/inotify/inotify_user.c)
```c
static ssize_t inotify_read(struct file *file, char __user *buf,
			    size_t count, loff_t *pos)
{
	struct fsnotify_group *group;
	struct fsnotify_event *kevent;
	char __user *start;
	int ret;
	DEFINE_WAIT_FUNC(wait, woken_wake_function);

	start = buf;
	group = file->private_data;

	add_wait_queue(&group->notification_waitq, &wait);
	while (1) {
		spin_lock(&group->notification_lock);
		kevent = get_one_event(group, count);
		spin_unlock(&group->notification_lock);

		pr_debug("%s: group=%p kevent=%p\n", __func__, group, kevent);

		if (kevent) {
			ret = PTR_ERR(kevent);
			if (IS_ERR(kevent))
				break;
			ret = copy_event_to_user(group, kevent, buf);
			fsnotify_destroy_event(group, kevent);
			if (ret < 0)
				break;
			buf += ret;
			count -= ret;
			continue;
		}

		ret = -EAGAIN;
		if (file->f_flags & O_NONBLOCK)
			break;
		ret = -ERESTARTSYS;
		if (signal_pending(current))
			break;

		if (start != buf)
			break;

		wait_woken(&wait, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
	}
	remove_wait_queue(&group->notification_waitq, &wait);

	if (start != buf && ret != -EFAULT)
		ret = buf - start;
	return ret;
}
```

## DEFINE_WAIT_FUNC
출처: [https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/include/linux/wait.h](https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/include/linux/wait.h)
```c
#define DEFINE_WAIT_FUNC(name, function)					\
	struct wait_queue_entry name = {					\
		.private	= current,					\
		.func		= function,					\
		.entry		= LIST_HEAD_INIT((name).entry),			\
	}
```
`DEFINE_WAIT_FUNC` 매크로는 대기열 (wait_queue) 안에 들어갈 `wait_queue_entry` 구조체를 선언하고 있다. 이 매크로는 `private`, `func`, `entry` 세 개의 필드를 초기화한다. `private` 필드에는 현재 태스크의 `task_struct`가, `func` 필드에는 깨어난 직후 실행할 함수가 (이 경우에는 `woken_wake_func`), `entry`에는 [이중 연결 리스트 구조체](linked-list-of-linux-kernel.md)가 들어간다.

## woken_wake_func
출처: [https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/kernel/sched/wait.c](https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/kernel/sched/wait.c)
```c
int woken_wake_function(struct wait_queue_entry *wq_entry, unsigned mode, int sync, void *key)
{
	/* Pairs with the smp_store_mb() in wait_woken(). */
	smp_mb(); /* C */
	wq_entry->flags |= WQ_FLAG_WOKEN;

	return default_wake_function(wq_entry, mode, sync, key);
}
```
`woken_wake_function`은 `wait_queue_entry`의 `WQ_FLAG_WOKEN`플래그 비트를 1로 변경하고 `default_wake_function`을 호출한다. 이 플래그는 태스크가 방금 막 깨어났다는 것을 의미한다.

출처: [https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/kernel/sched/core.c](https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/kernel/sched/core.c)
```c
int default_wake_function(wait_queue_entry_t *curr, unsigned mode, int wake_flags, void *key)
{
	WARN_ON_ONCE(IS_ENABLED(CONFIG_SCHED_DEBUG) && wake_flags & ~(WF_SYNC|WF_CURRENT_CPU));
	return try_to_wake_up(curr->private, mode, wake_flags);
}
```
`default_wake_function`은 `try_to_wake_up`을 실행시키고, 이 함수는 `ttwu` 함수들을 호출해 태스크를 이전에 실행되던 프로세서의 runqueue에 다시 삽입한다. 
## add_wait_queue
이렇게 만들어진 `wait_queue_entry`를 파일 기술자의 대기 큐에 삽입한 다음, 태스크는 `while(1)` 루프에 진입한다. 이 루프 안에서 태스크는 `wait_woken`을 호출해 이벤트가 발생할 때까지 대기 상태에 들어간다.
## 인터럽트 발생
인터럽트가 발생되면 가장 먼저 `woken_wake_function`이 실행되고, 깨어난 태스크 `get_one_event` 함수를 통해 이벤트를 가져오려고 시도한다. 이벤트를 가져오는 데 성공한 경우, 가져온 이벤트를 사용자 메모리에 복사하고 `remove_wait_queue`를 사용해 대기열에서 자신의 `wait_queue_entry`를 삭제해 대기를 종료한다. 만약 이벤트를 가져오지 못했다면 `wait_woken` 함수를 실행해 다시 대기 상태에 들어간다.

## wait_woken
출처: [https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/kernel/sched/wait.c](https://github.com/torvalds/linux/blob/1acfd2bd3f0d9dc34ea1871a445c554220945d9f/kernel/sched/wait.c)
```c
long wait_woken(struct wait_queue_entry *wq_entry, unsigned mode, long timeout)
{
	/*
	 * The below executes an smp_mb(), which matches with the full barrier
	 * executed by the try_to_wake_up() in woken_wake_function() such that
	 * either we see the store to wq_entry->flags in woken_wake_function()
	 * or woken_wake_function() sees our store to current->state.
	 */
	set_current_state(mode); /* A */
	if (!(wq_entry->flags & WQ_FLAG_WOKEN) && !kthread_should_stop_or_park())
		timeout = schedule_timeout(timeout);
	__set_current_state(TASK_RUNNING);

	/*
	 * The below executes an smp_mb(), which matches with the smp_mb() (C)
	 * in woken_wake_function() such that either we see the wait condition
	 * being true or the store to wq_entry->flags in woken_wake_function()
	 * follows ours in the coherence order.
	 */
	smp_store_mb(wq_entry->flags, wq_entry->flags & ~WQ_FLAG_WOKEN); /* B */

	return timeout;
}
```

`wait_woken` 함수는 가장 먼저 `set_current_state` 함수가 실행하고, 현재 태스크의 상태를 `TASK_INTERRUPTIBLE` 또는 `TASK_UNINTERRUPTIBLE` 상태로 바꾸어 대기 상태로 들어가게 한다.

`!(wq_entry->flags & WQ_FLAG_WOKEN) && !kthread_should_stop_or_park()` 표현식은 태스크가 대기열 안에서 대기하고 있는 동안 다른 태스크의 호출로 인해 활성화되었을 경우 참이 된다. 이럴 경우 다른 태스크에게 프로세서를 양보하고 `timeout`만큼 다시 대기한다.

인터럽트에 의해 깨어났을 경우, `__set_current_state` 함수를 호출해 태스크의 상태를 `TASK_RUNNING`으로 변경한다. 함수명 앞의 `__`는 이 함수를 `smp_mb`로 보호받고 있는 안전한 환경에서만 실행해야 한다는 것을 의미한다.

그 후, `smp_store_mb`함수를 호출해 `wait_queue_entry`의 `WQ_FLAG_WOKEN`비트를 0으로 변경한다. 이렇게 태스크를 깨우는 프로세스가 끝나고, 실행된 태스크는 `while(1)` 루프 안에서 실행 조건을 검사해 처리를 계속할지, 아니면 다시 대기할지를 결정하게 된다.

# 정리
실행 과정을 다시 정리해 보면 다음과 같다.
```
inotify_read -> add_wait_queue -> wait_woken -> woken_wake_func -> ttwu -> 실행 조건 검사 -> wait_woken / remove_wait_queue
```

이 과정에서 `WQ_FLAG_WOKEN` 비트의 변화는 다음과 같다.

| call | inotify_read | add_wait_queue | wait_woken | woken_wake_func | inotify_read | remove_wait_queue |
| ---- | ---- | ---- | ---- | ---- | ---- | --- |
| WQ_FLAG_WOKEN | 0 | 0 | 0 | 1 | 1 | 0 | 0 | 0 |

# 실습
```c
#include <stdio.h>  
#include <sys/inotify.h>  
#include <unistd.h>  
  
#define EVENT_SIZE  (sizeof(struct inotify_event))  
#define BUF_LEN     (1024 * (EVENT_SIZE + 16))  
  
int main(void){  
    int fd = inotify_init();  
    int wd = inotify_add_watch(fd, ".", IN_MODIFY|IN_CREATE|IN_DELETE);  
  
    int pid = fork();  
    printf("pid %d created\n", pid);  
  
    char buffer[BUF_LEN];  
  
    printf("pid %d before_read\n", pid);  
    int len = read(fd, buffer, BUF_LEN);  
    printf("pid %d after_read\n", pid);  
  
    for (int i=0; i<len;){  
        struct inotify_event *event = (struct inotify_event *) &buffer[i];  
        if (event->len){  
            if (event->mask & IN_CREATE) {  
                printf("The file %s was created.\n", event->name);  
            } else if (event->mask & IN_DELETE) {  
                printf("The file %s was deleted.\n", event->name);  
            } else if (event->mask & IN_MODIFY) {  
                printf("The file %s was modified.\n", event->name);  
            }  
        }  
        i += EVENT_SIZE + event->len;  
    }  
}
```

동일한 대기 큐를 두 개의 태스크가 공유하도록 하기 위해 `inotify_init`으로 파일 기술자를 가져온 후 `fork`를 수행하여 동일한 파일 기술자에 두 번의 `read`를 수행하게 하였다.

```
pid 1028216 created
pid 1028216 before_read
pid 0 created
pid 0 before_read
pid 0 after_read
The file test was created.
pid 1028216 after_read
The file test was deleted.
```

테스트 결과 첫 번째 inotify 이벤트에서는 대기 큐의 가장 처음에 위치한 (=가장 마지막에 들어간) 자식 태스크가 실행되었으며 두 번째 이벤트에서는 부모 태스크가 실행된 것을 확인할 수 있었다.
# 참고문헌
* [https://stackoverflow.com/questions/13351172/inotify-file-in-c](https://stackoverflow.com/questions/13351172/inotify-file-in-c)