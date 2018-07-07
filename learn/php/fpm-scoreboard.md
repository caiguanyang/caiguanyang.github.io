[TOC]

# fpm_scoreboard学习

参考：https://blog.csdn.net/mijar2016/article/details/53414402
=======
## 作用

用于记录worker进程运行信息的结构，此结构分配在共享内存上，master通过这种方式获取worker的运行信息；

## 关键数据结构

```c
struct fpm_worker_pool_s {
  int listening_socket;
  struct fpm_scoreboard_s *scoreboard;
  struct fpm_child_s *children;
  .....
}
struct fpm_child_s {
  struct fpm_child_s *prev, *next;
  struct fpm_worker_pool_s *wp;
  int scoreboard_i;
  ....
}
struct fpm_scoreboard_s {
  int pm;
  int idle;
  int active;
  int active_max;
  struct fpm_scoreboard_proc_s *procs[];
  .....
}
struct fpm_scoreboard_proc_s {
  pid_t pid;
  struct timeval accepted;
  struct timeval duration;
  enum fpm_request_stage_e request_stage;
  .....
}
```


## fpm_scoreboard初始化流程

main()-->fpm_init()-->fpm_scoreboard_init_main()

```c
// fpm_scoreborad.c : fpm_scoreboard_init_main()
int fpm_scoreboard_init_main() {
  struct fpm_worker_pool_s *wp;
  for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
        scoreboard_size        = sizeof(struct fpm_scoreboard_s) + (wp->config->pm_max_children) * sizeof(struct fpm_scoreboard_proc_s *);
		scoreboard_nprocs_size = sizeof(struct fpm_scoreboard_proc_s) * wp->config->pm_max_children;
        // 共享内存mmap
		shm_mem                = fpm_shm_alloc(scoreboard_size + scoreboard_nprocs_size);

		if (!shm_mem) {
			return -1;
		}
		wp->scoreboard         = shm_mem;
		wp->scoreboard->nprocs = wp->config->pm_max_children;
		shm_mem               += scoreboard_size;

		for (i = 0; i < wp->scoreboard->nprocs; i++, shm_mem += sizeof(struct fpm_scoreboard_proc_s)) {
			wp->scoreboard->procs[i] = shm_mem;
		}

		wp->scoreboard->pm          = wp->config->pm;
		wp->scoreboard->start_epoch = time(NULL);
		strlcpy(wp->scoreboard->pool, wp->config->name, sizeof(wp->scoreboard->pool));
  }
  return 0;
}
```


## scoreboard数据更新流程

## master进程读取scoreboard数据

# FPM源码结构
+ fpm_request.c
  worker进程处理请求的逻辑
  包含请求的各个状态
```
enum fpm_request_stage_e {
	FPM_REQUEST_ACCEPTING = 1,
	FPM_REQUEST_READING_HEADERS,
	FPM_REQUEST_INFO,
	FPM_REQUEST_EXECUTING,
	FPM_REQUEST_END,
	FPM_REQUEST_FINISHED
};
```
=======
scoreboard结构的数据都是由master进程更新的，而更新的时机有如下两点

1）worker进程创建：

   执行逻辑：

​     master逻辑：fpm_children_make() --> fpm_resources_prepare() --> fpm_scoreboard_proc_alloc();

​     worker逻辑：fpm_children_make() -->fpm_child_resources_use() --> fpm_scoreboard_child_use()

2）worker进程销毁：master进程接收到SIGCHLD后，执行fpm_children_bury()-->fpm_scoreboard_proc_free(wp->scoreboard, child->scoreboard_i) 逻辑；

3）ondemand模式下，接收到客户端连接时，执行IO事件处理函数

注册事件：fpm_run() --> fpm_children_create_initial() 

IO处理函数逻辑：fpm_pctl_on_socket_accept() --> fpm_scoreboard_update() --> fpm_children_make(wp, 1, 1, 1)

 4）static或者dynamic模式下worker进程的新建或销毁



## 问题

1）worker进程创建后，进入到worker进程的逻辑，执行的第一步为fpm_child_resources_use，但为什么要释放其他worker_pool的scoreboard数据呢？

需要再了解下mmap，以及子进程是否继承这些数据

```c
// fpm_children.c
static void fpm_child_resources_use(struct fpm_child_s *child) /* {{{ */
{
    // 为什么要释放其他资源
	struct fpm_worker_pool_s *wp;
	for (wp = fpm_worker_all_pools; wp; wp = wp->next) {
		if (wp == child->wp) {
			continue;
		}
		fpm_scoreboard_free(wp->scoreboard);
	}

	fpm_scoreboard_child_use(child->wp->scoreboard, child->scoreboard_i, getpid());
	fpm_stdio_child_use_pipes(child);
	fpm_child_free(child);
}

```




