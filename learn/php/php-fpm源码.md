# php-fpm源码--fpm_main.c

[TOC]

任务：

+ accept惊群问题
+ worker进程执行逻辑
+ master进程和worker进程如何通信？如kill，worker进程信息同步
+ scoreboard学习
+ ​



php v7.0.12

## 主逻辑



```c
int main() {
  sapi_startup(&cgi_sapi_module);
  if (cgi_sapi_module.startup(&cgi_sapi_module) == FAILURE) {
  	return FPM_EXIT_SOFTWARE;
  }
  fcgi_init();
  fpm_init(....);
  // 开始启动master && worker进程
  fpm_is_running = 1;
  fcgi_fd = fpm_run(&max_requests);
  parent = 0;
  // worker进程处理逻辑
  ....
}
```

## php生命周期

http://xingdong365.com/program/345.html



## sapi_startup流程

参考文件/src/main/SAPI.c

### cgi_sapi_module

```c
// src/fpm/fpm/fpm_main.c : 846
static sapi_module_struct cgi_sapi_module = {
	"fpm-fcgi",						/* name */
	"FPM/FastCGI",					/* pretty name */

	php_cgi_startup,				/* startup */
	php_module_shutdown_wrapper,	/* shutdown */

	sapi_cgi_activate,				/* activate */
	sapi_cgi_deactivate,			/* deactivate */
    .....
}

// src/main/SAPI.h : 220
typedef struct _sapi_module_struct {
	char *name;
	char *pretty_name;

	int (*startup)(struct _sapi_module_struct *sapi_module);
	int (*shutdown)(struct _sapi_module_struct *sapi_module);

	int (*activate)(void);
	int (*deactivate)(void);
	.....
} sapi_module_struct;
```

### php_cgi_startup

进入php生命周期module startup阶段

```c
// src/main/php_main.h : 33
PHPAPI int php_module_startup(sapi_module_struct *sf, zend_module_entry *additional_modules, uint num_additional_modules);
```

+ src/main/php_main.h包含PHP内核代码运行的关键函数声明



## fcgi_init流程

参考/src/main/fastcgi.h



## fpm_init流程

参考/src/sapi/fpm/fpm/fpm.h

其中fpm_{module}_init_main()函数在fpm_{module}.h文件中定义

```c
// src/sapi/fpm/fpm/fpm.c : 46
if (0 > fpm_php_init_main()           ||
	    0 > fpm_stdio_init_main()         ||
	    0 > fpm_conf_init_main(test_conf, force_daemon) ||
	    0 > fpm_unix_init_main()          ||
	    0 > fpm_scoreboard_init_main()    ||
	    0 > fpm_pctl_init_main()          ||
	    0 > fpm_env_init_main()           ||
	    0 > fpm_signals_init_main()       ||
	    0 > fpm_children_init_main()      ||
	    0 > fpm_sockets_init_main()       ||
	    0 > fpm_worker_pool_init_main()   ||
	    0 > fpm_event_init_main()) {
  .....
}
struct fpm_globals_s {
	pid_t parent_pid;
	int argc;
	char **argv;
	char *config;
	char *prefix;
	char *pid;
	int running_children;
	int error_log_fd;
	int log_level;
	int listening_socket; /* for this child */
	int max_requests; /* for this child */
	int is_child;
	int test_successful;
	int heartbeat;
	int run_as_root;
	int force_stderr;
	int send_config_pipe[2];
};
```



## fpm_run流程



```c
// src/fpm/fpm/fpm.h : 91	
/*	children: return listening socket
	parent: never return */
int fpm_run(int *max_requests) /* {{{ */
{
	struct fpm_worker_pool_s *wp;

	/* create initial children in all pools */
	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
		int is_parent;
		// worker-进程初始化：监听listen-fd，创建worker进程
        //    fpm_children_make()
		is_parent = fpm_children_create_initial(wp);

		if (!is_parent) {
			goto run_child;
		}

		/* handle error */
		if (is_parent == 2) {
			fpm_pctl(FPM_PCTL_STATE_TERMINATING, FPM_PCTL_ACTION_SET);
			fpm_event_loop(1);
		}
	}

  	// master-进程进入fpm_event_loop，不再退出
	/* run event loop forever */
	fpm_event_loop(0);

    // worker-进程则返回监听的socket
run_child: /* only workers reach this point */
	fpm_cleanups_run(FPM_CLEANUP_CHILD);
	*max_requests = fpm_globals.max_requests;
	return fpm_globals.listening_socket;
}
/* }}} */
```

### master进程

​    fpm_run中进入到fpm_event_loop中

```c
// src/fpm/fpm_events.c : 346
void fpm_event_loop(int err) /* {{{ */
{	
	static struct fpm_event_s signal_fd_event;

    fpm_event_set(&signal_fd_event, fpm_signals_get_fd(), FPM_EV_READ, &fpm_got_signal, NULL);
	fpm_event_add(&signal_fd_event, 0);
    while(1) {
  		/* 1s timeout if none has been set */
		if (!timerisset(&ms) || timercmp(&ms, &now, <) || timercmp(&ms, &now, ==)) {
			timeout = 1000;
		} else {
			timersub(&ms, &now, &tmp);
			timeout = (tmp.tv_sec * 1000) + (tmp.tv_usec / 1000) + 1;
		}	
		ret = module->wait(fpm_event_queue_fd, timeout);
      
        q = fpm_event_queue_timer;
		while (q) {
			fpm_clock_get(&now);
          	// 调用事件处理函数
         	fpm_event_fire(q->ev); 
            ....
        }
    }
}
// 事件信息
struct fpm_event_s {
	int fd;                   /* not set with FPM_EV_TIMEOUT */
	struct timeval timeout;   /* next time to trigger */
	struct timeval frequency;
	void (*callback)(struct fpm_event_s *, short, void *);
	void *arg;
	int flags;
	int index;                /* index of the fd in the ufds array */
	short which;              /* type of event */
};
```



### worker进程



```c
// src/fpm/fpm_worder_pool.h
struct fpm_worker_pool_s {
	struct fpm_worker_pool_s *next;
	struct fpm_worker_pool_config_s *config;
	char *user, *home;			/* for setting env USER and HOME */
	enum fpm_address_domain listen_address_domain;
	int listening_socket;
	int set_uid, set_gid;		/* config uid and gid */
	int socket_uid, socket_gid, socket_mode;
	/* runtime */
	struct fpm_child_s *children;
	int running_children;
	int idle_spawn_rate;
	int warn_max_children;
#if 0
	int warn_lq;
#endif
	struct fpm_scoreboard_s *scoreboard;
	int log_fd;
	char **limit_extensions;
	/* for ondemand PM */
	struct fpm_event_s *ondemand_event;
	int socket_event_set;
#ifdef HAVE_FPM_ACL
	void *socket_acl;
#endif
};
```



## 进程间通信

### 信号处理流程

+ master进程收到终止信号的处理流程（SIGTERM）

fpm_signal.c中的sig_handler将捕获的信号写入到unix管道中，那么具体的信号处理函数是哪个文件呢？它如何读取信号？

​       fpm_signal.c中的fpm_signals_get_fd函数返回管道描述符，在fpm_events.c中的fpm_event_loop中添加到事件管理器中，fpm_got_signal函数从管道中读取信号值，调用fpm_process_ctl.c中的fpm_pctl函数处理。

### master进程如何读取worker进程的状态






## FPM事件管理器分析

参考: https://www.jianshu.com/p/e567ba80f3b2

​         可以参考下他写的其他文章

```c
// src/fpm/fpm_events.c : 47
typedef struct fpm_event_queue_s {
	struct fpm_event_queue_s *prev;
	struct fpm_event_queue_s *next;
	struct fpm_event_s *ev;
} fpm_event_queue;
// 超时事件和fd事件放到两个queue中
static struct fpm_event_queue_s *fpm_event_queue_timer = NULL;
static struct fpm_event_queue_s *fpm_event_queue_fd = NULL;

// 抽象的event描述符，用于兼容select, poll, epoll
struct fpm_event_s {
	int fd;                   /* not set with FPM_EV_TIMEOUT */
	struct timeval timeout;   /* next time to trigger */
	struct timeval frequency;
	void (*callback)(struct fpm_event_s *, short, void *);
	void *arg;
	int flags;
	int index;                /* index of the fd in the ufds array : poll, select*/
	short which;              /* type of event */
};
```



### master进程事件

master进程在event_loop中等待事件发生(epoll_wait)，并遍历链表中的定时器事件(fpm_event_queue_timer)

+ 定时事件

```c
// fpm_events.c : fpm_event_loop()
fpm_pctl_heartbeat(NULL, 0, NULL);
fpm_pctl_perform_idle_server_maintenance_heartbeat(NULL, 0, NULL);
fpm_systemd_heartbeat(NULL, 0, NULL);
```

​     反复检查，到时逐个串行运行；



+ IO事件：控制信号处理

​       socketpair创建了一个管道，不过并非用于进程间通信，只是用来接收用户发起的命令，如SIGINT, SIGTERM

```c
// fpm_events.c : fpm_event_loop()
fpm_event_set(&signal_fd_event, fpm_signals_get_fd(), FPM_EV_READ, &fpm_got_signal,   NULL);
fpm_event_add(&signal_fd_event, 0);
```

+ IO事件：PM_STYLE_ONDEMAND进程管理策略，监听worker进程的个数；当接收到新请求后才会触发

为每个work_pool添加一个IO事件

```c
// fpm_events.c : fpm_children_create_initial()
memset(wp->ondemand_event, 0, sizeof(struct fpm_event_s));
fpm_event_set(wp->ondemand_event, wp->listening_socket, FPM_EV_READ | FPM_EV_EDGE,  fpm_pctl_on_socket_accept, wp);
wp->socket_event_set = 1;
fpm_event_add(wp->ondemand_event, 0);
```



### worker进程事件

+ 监听listen-socket 




```c

```



+ ​




## 问题

### listen socket由谁创建

master进程在fpm_init的过程中通过fpm_sockets_init_main创建

```c
// fpm_sockets.c : fpm_sockets_init_main()
switch (wp->listen_address_domain) {
			case FPM_AF_INET :
				wp->listening_socket = fpm_socket_af_inet_listening_socket(wp);
				break;

			case FPM_AF_UNIX :
				if (0 > fpm_unix_resolve_socket_premissions(wp)) {
					return -1;
				}
				wp->listening_socket = fpm_socket_af_unix_listening_socket(wp);
				break;
}	
```

### worker进程如何accept连接

worker进程在fpm_run中执行主要的逻辑

```c
// php-src/sapi/fpm/fpm_main.c : main()
...
fcgi_fd = fpm_run(&max_requests);
...
request = fpm_init_request(fcgi_fd);
// 每个worker进程accept请求
while (EXPECTED(fcgi_accept_request(request) >= 0)) {
  SG(server_context) = (void *) request;
  init_request_info();
  fpm_request_info();
  php_request_startup();
  fpm_status_handle_request();
  fpm_php_limit_extensions(SG(request_info).path_translated);
  php_fopen_primary_script(&file_handle);
  fpm_request_executing();
  // php-src/main/main.c
  php_execute_script(&file_handle);
// goto标志
fastcgi_request_done:
  ...
  fpm_request_end();
  fpm_log_write(NULL);
  php_request_shutdown((void *) 0);
  // 如果执行的请求次数大于约定阈值，则退出进程，参考说明
  requests++;
  if (UNEXPECTED(max_requests && (requests == max_requests))) {
	fcgi_finish_request(request, 1);
	break;
  }
} // end of loop
fcgi_destroy_request(request);
fcgi_shutdown();
...
```

**worker进程执行约定个请求后退出说明**

```php
// php-fpm配置
How much requests each process should execute before respawn.
Useful to work around memory leaks in 3rd party libraries.
For endless request processing please specify 0
Equivalent to PHP_FCGI_MAX_REQUESTS
<value name="max_requests">10000</value>
```

### 多个worker进程如何处理accept惊群

通过锁机制，保证只有一个进程执行accept操作。

Linux内核v2.6之后已解决accept惊群问题

参考：https://www.cnblogs.com/Anker/p/7071849.html

```c
// php-src/main/fastcgi.c : fcgi_accept_request()
int fcgi_accept_request(fcgi_request *req){
  while (1) {  // while-1
    if (req->fd < 0) {
  	  while (1) {  // while-2
  		if (in_shutdown) {
		  return -1;
		}
        req->hook.on_accept();
        // 通过锁处理accept惊群
		FCGI_LOCK(req->listen_socket);
		req->fd = accept(listen_socket, (struct sockaddr *)&sa, &len);
		FCGI_UNLOCK(req->listen_socket);
        
        if (req->fd >= 0 && !fcgi_is_allowed()) {
  		  closesocket(req->fd);
          req->fd = -1;
          continue;
		}
      } // end of while-2 
    } else if (in_shutdown) {
	  return -1;
	}
    // 解析Fastcgi协议
    if (fcgi_read_request(req)) {
      return req->fd;
    } else {
  	  fcgi_close(req, 1, 1);
    }
  } // end of while-1
}
```

有关请求读取参考https://www.cnblogs.com/sunlong88/p/9001184.html



### worker进程的创建规则

+ static模式：启动时master进程根据配置创建max_children个worker进程，固定不变；
+ dynamic模式：服务过程中动态调整worder进程的个数[min_spare_servers, max_children]，其他参考参数start_servers, max_spare_servers;
+ ondemand模式：启动时不创建worker进程，来请求后才会fork进程，但请求结束后不会立即退出，空闲时间超过process_idle_timeout后再退出，总的worker进程不超过max_children个。

```c
// fpm_children.c : fpm_children_create_initial()
if (wp->config->pm == PM_STYLE_ONDEMAND) {
  wp->ondemand_event = (struct fpm_event_s *)malloc(sizeof(struct fpm_event_s));
  memset(wp->ondemand_event, 0, sizeof(struct fpm_event_s));
  fpm_event_set(wp->ondemand_event, wp->listening_socket, FPM_EV_READ | FPM_EV_EDGE, fpm_pctl_on_socket_accept, wp);
  wp->socket_event_set = 1;
  fpm_event_add(wp->ondemand_event, 0);
}
```

事件处理函数

```c
// fpm_process_ctl.c :　fpm_pctl_on_socket_accept()
void fpm_pctl_on_socket_accept(struct fpm_event_s *ev, short which, void *arg) /* {{{ */
{
	struct fpm_worker_pool_s *wp = (struct fpm_worker_pool_s *)arg;
	struct fpm_child_s *child;
    ...
    wp->warn_max_children = 0;
	fpm_children_make(wp, 1, 1, 1);
	if (fpm_globals.is_child) {
		return;
	}
}
```



+ worker进程中有一个退出机制，对于static模式，如何支持？

```c
// fpm_main.c : main()  
  // max_request在php-fpm配置文件中设置
  requests++;
  if (UNEXPECTED(max_requests && (requests == max_requests))) {
	fcgi_finish_request(request, 1);
	break;
  }
```

master进程接收到worker进程退出的信号(SIGCHILD)后会执行fpm_children_bury()函数，它会判断进程是否需要重启，static模式通过此方式可用于重启进程。

**疑问?????????：** 

但是通过event_loop中的fpm_got_signal信号处理函数、或者IO事件回调创建worker进程后，执行fpm_child_init后就退出了呢，后续执行什么操作呢？



### php-fpm为什么采用多进程模型

以及采用阻塞的方式处理请求，限制了并发？

https://www.jianshu.com/p/30b494f861e2



+ ​



## 文件结构

+ ​
+ fpm_main.c

fpm的入口文件；调用init和run函数，并执行worker进程的主逻辑

+ fpm.h

定义fpm_init()和fpm_run()函数；

fpm_init: 调用各子模块的fpm_ ?? _init_main函数，完成fpm的初始化工作

fpm_run: 创建worker进程；master进程进入fpm_event_loop() 

+ fpm_children.h

主要负责worker进程的管理，如初始化、创建、资源分配/回收等；

+ fpm_scoreboard.h

用于记录worker进程的工作状态

+ fpm_signals.h

master进程和worker进程注册信号处理函数；

master进程创建一个管道，用于注册IO事件到事件管理器，监听用户命令输入

+ fpm_events.h

fpm事件管理器，提供抽象结构用于支持不同类型IO多路复用，select, poll, epoll

fpm_event_loop()：master进程主逻辑

+ fpm_cleanup.h

​       用fpm_array_s存储各子模块的cleanup函数，用于资源回收

+ fpm_request.h


worker进程处理http请求的工具函数集，主要包含如下几个状态：FPM_REQUEST_ACCEPTING、FPM_REQUEST_READING_HEADERS、FPM_REQUEST_INFO、FPM_REQUEST_EXECUTING、FPM_REQUEST_END、FPM_REQUEST_FINISHED

+ fpm_worker_pool.h


worker_pool结构体的维护，采用全局变量的方式（fpm_worker_all_pools），多个进程共享访问此结构中的数据，如socoreboard；通过fpm_conf_init_main完成worker_pool的初始化。

+ ​



## 参考

https://www.jianshu.com/p/542935a3bfa8






## linux基础

### Signal

信号注册

信号发送

处理函数

signal(int signo, handler)

kill(pid_t, int signo)

#### 信号屏蔽和恢复

sigempty、sigaddset、sigaction、sigprocmask

信号屏蔽机制可用于实现关键代码运行不被中断；



#### SIGPIPE

参考：https://www.cnblogs.com/caosiyang/archive/2012/07/19/2599071.html

连接建立后，如果一端断开连接，另一端仍旧相对方写数据，第一次写数据收到RST响应，此后再写数据，则内核向进程发起SIGPIPE信号，通知进程连接已断开。

```c
// line:1599
#ifdef HAVE_SIGNAL_H
#if defined(SIGPIPE) && defined(SIG_IGN)
	signal(SIGPIPE, SIG_IGN); /* ignore SIGPIPE in standalone mode so
								that sockets created via fsockopen()
								don't kill PHP if the remote site
								closes it.  in apache|apxs mode apache
								does that for us!  thies@thieso.net
								20000419 */
#endif
#endif
```



### 进程创建

#### fork

参考：https://www.cnblogs.com/qiuheng/p/5752284.html

### IO多路复用

**epoll:**

   https://blog.csdn.net/men_wen/article/details/53456491

```c
#include<sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);

struct epoll_event{
  uint32_t events;
  epoll_data_t data;
};
typedef union epoll_t {
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

  Level-Trigger 和 Edge-Trigger

**poll:**

```c
#include<poll.h>
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
```

**pselect:**

```c
#include<sys/select.h>
int pselect(int maxfdpl, fd_set *restrict readfds, fd_set *restrict writefds,
           fd_set *restrict exceptfds,
           const struct timespec *restrict tsptr,
           const sigset_t *restrict sigmask);
```

**select:**

```c
#include<sys/select>
int select(int maxfdpl, fd_set *restrict readfds, fd_set *restrict writefds,
          fd_set *restrict exceptfds,
          struct timeval *restrict tvptr);
// fd_set相关操作
int FD_ISSET(int fd, fd_set *fdset);
int FD_CLR(int fd, fd_set *fdset);
int FD_SET(int fd, f_set *fdset);
int FD_ZERO(fd_set *fdset);   
```

+ select和pselect的区别
+ select和poll的区别
+ epoll和poll的区别

### 进程间通信--管道



```c
// fpm/fpm_signal.c : 182
static int sp[2];
int fpm_signals_init_main() {
  if (0 > socketpair(AF_UNIX, SOCK_STREAM, 0, sp)) {
		zlog(ZLOG_SYSERROR, "failed to init signals: socketpair()");
		return -1;
  }
}
// linux 匿名管道
#include <unistd.h>
int pipe(int fd[2]);
```





### 进程间通信--共享内存







