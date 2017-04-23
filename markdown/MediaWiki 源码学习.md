# MediaWiki 源码学习





配置项

改动备注：

include/Sanitizer.php:870

   注释掉 preg_replace_callback()调用

include/db/Database.php:3164

​	添加了 empty 判断





## 关键文件说明

wfDebugLog()  日志打印到什么地方了



### LocalSetting.php

mediawiki配置文件。

配置项说明参考：http://blog.csdn.net/fjgysai/article/details/2255246



### WebStart.php

参考：http://blog.csdn.net/fjgysai/article/details/2255239 

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

function performRequest() {
  $article = $this->initializeArticle();
  if (is_object($article)) {
    $this->perfromAction($article, $reqeustTitle);
    
  } else if (is_string($article)) {
    // 重定向
    $output->redirect($article);
  } else {
    throw new Exception();
  }
}

function initializeArticle() {
  if ($this->context->canUseWikiPage()) {
    $apge = $this->context->getWikiPage();
    $article = Article::newFromWikiPage($page, $this->context);
  } else {
    $article = Article::newFromTitle($title, $this->context);
    $this->context->setWikiPage($article->getPage());
  }
}

function performAction(Page $page, Title $requestTitle) {
  $act = $this->getAction();
  $action = Action::factory($act, $page, $this->context);
  $action->show();
}

```



### Article.php

​	依赖 WikiPage.php

```php
// This is the default action of the index.php entry point: just view the page of the given title.
public function view() {
  
}
```





### actions/???Action.php

处理结果





### GlobalFunctions.php

定义了全局函数，如 wfGetLB()  wfGetDB() ,wfDebug() 等



### 日志打印

include/debug/Debug.php











