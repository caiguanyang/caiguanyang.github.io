[TOC]

# fpm_scoreboard学习
参考：https://blog.csdn.net/mijar2016/article/details/53414402

## fpm_scoreboard初始化流程

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
