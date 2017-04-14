# MediaWiki 源码学习



改动备注：

include/Sanitizer.php:870

   注释掉 preg_replace_callback()调用

include/db/Database.php:3164

​	添加了 empty 判断





## 关键文件说明

wfDebugLog()  日志打印到什么地方了







### WebStart.php

初始化一个请求的环境

include 一推公共文件

​      include/AutoLoader.php

​     include/vendor/autoload.php

​     include/Defines.php

​    include/DefaultSettings.php

如下函数的作用：？？？？？

wfProfileOut()

wfProfileIn()

看下 cache吧

include/cache/MessageCache.php:326

  

### Wiki.php

定义类 MediaWiki

```php
function run() {
  $this->checkMaxLag();
  $this->main();
  $this->restInPeace();
}

function main() {
  if ($wgUseAjax) {
    $dispatcher = new AjaxDispatcher();
    $dispatcher->performAction();
    return;
  }
  // 处理 https
  if ($reqeust->getCookie('forceHTTPS')) {
    $redirUrl = str_replace('http://', 'https://', $oldUrl);
    $output->redirect($redirUrl);
  }
  // user file cache
  if ($wgUseFileCache) {
    ...
  }
  $this->performRequest()；
}

function performRequest() {}


```





### GlobalFunctions.php

定义了全局函数，如 wfGetLB()  wfGetDB() ,wfDebug() 等



### 日志打印

include/debug/Debug.php











