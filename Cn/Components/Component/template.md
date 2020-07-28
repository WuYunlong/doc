---
title: easyswoole Http服务-模版引擎
meta:
  - name: description
    content: easyswoole Http服务-模版引擎
  - name: keywords
    content: easyswoole Http服务-模版引擎
---

# 模板引擎

## 渲染驱动
`EasySwoole`引入模板渲染驱动的形式，把需要渲染的数据，通过协程客户端投递到自定义的同步进程中进行渲染并返回结果。为何要如此处理，原因在于，市面上的一些模板引擎在`Swoole`协程下存在变量安全问题。例如以下流程：
   
   - request A reached, static A assign requestA-data
   - compiled template 
   - write compiled template (yiled current coroutine)
   - request B reached，static A assign requestB-data
   - render static A data into complied template file
   
   以上流程我们可以发现，A请求的数据，被B给污染了。为了解决该问题，`EasySwoole`引入模板渲染驱动模式。

## 安装

> composer require easyswoole/template


## 基础实现讲解

### 实现渲染引擎
```php
use EasySwoole\Template\Config;
use EasySwoole\Template\Render;
use EasySwoole\Template\RenderInterface;

class R implements RenderInterface
{

    public function render(string $template, array $data = [], array $options = []):?string
    {
        return 'todo some thing';
    }

    public function afterRender(?string $result, string $template, array $data = [], array $options = [])
    {
        // TODO: Implement afterRender() method.
    }

    public function onException(Throwable $throwable):string
    {
        return $throwable->getMessage();
    }
}

```  

### 自定义HTTP服务中调用

```php
\EasySwoole\Template\Render::getInstance()->getConfig()->setRender(new R());

$httpServer = new \Swoole\Http\Server("0.0.0.0", 9501);
$httpServer->on("request", function (\Swoole\Http\Request $request, \Swoole\Http\Response $response)use($render) {
    //调用渲染器，此时会通过携程客户端，把数据发往自定义的同步进程中处理，并得到渲染结果
    $response->end(\EasySwoole\Template\Render::getInstance()->render('custom.html'));
});
$render->attachServer($httpServer);

$http->start();
```

### 重启渲染引擎
由于某些模板引擎会缓存模板文件
导致可能出现以下情况：
 × 用户A请求1.tpl 返回‘a’
 × 开发者修改了1.tpl的数据，改成了‘b’
 × 用户B，C，D在之后的请求中，可能会出现‘a’，‘b’两种不同的值
 
那是因为模板引擎已经缓存了A所在进程的文件，导致后面的请求如果也分配到了A的进程，就会获取到缓存的值

解决方案如下：
- 1:重启`easyswoole`服务，即可解决
- 2:模板渲染引擎实现了重启方法`restartWorker`，直接调用即可

````
Render::getInstance()->restartWorker();
````
用户可根据自己的逻辑，自行调用`restartWorker`方法进行重启
例如在控制器新增reload方法：
````php
<?php
namespace App\HttpController;


use EasySwoole\Http\AbstractInterface\Controller;
use EasySwoole\Template\Render;

class Index extends Controller
{

    public function index()
    {
        $this->response()->write(Render::getInstance()->render('index.tpl',[
            'user'=>'easyswoole',
            'time'=>time()
        ]));
    }

    public function reload(){
        Render::getInstance()->restartWorker();
        $this->response()->write(1);
    }
}
````

## 示例

### Smarty 渲染

#### 引入Smarty

> composer require smarty/smarty

#### 实现渲染引擎
```php
use EasySwoole\Template\RenderInterface;

class Smarty implements RenderInterface
{

    private $smarty;
    function __construct()
    {
        $temp = sys_get_temp_dir();
        $this->smarty = new \Smarty();
        $this->smarty->setTemplateDir(EASYSWOOLE_ROOT.'/View/');
        $this->smarty->setCacheDir("{$temp}/smarty/cache/");
        $this->smarty->setCompileDir("{$temp}/smarty/compile/");
    }

    public function render(string $template, array $data = [], array $options = []): ?string
    {
        foreach ($data as $key => $item){
            $this->smarty->assign($key,$item);
        }
        return $this->smarty->fetch($template,$cache_id = null, $compile_id = null, $parent = null, $display = false,
            $merge_tpl_vars = true, $no_output_filter = false);
    }

    public function afterRender(?string $result, string $template, array $data = [], array $options = [])
    {

    }

    public function onException(\Throwable $throwable): string
    {
        $msg = "{$throwable->getMessage()} at file:{$throwable->getFile()} line:{$throwable->getLine()}";
        trigger_error($msg);
        return $msg;
    }
}
```

#### HTTP服务中调用

在全局`mainServerCreate`事件中注册：

```php
\EasySwoole\Template\Render::getInstance()->getConfig()->setRender(new Smarty());
\EasySwoole\Template\Render::getInstance()->getConfig()->setTempDir(EASYSWOOLE_TEMP_DIR);
\EasySwoole\Template\Render::getInstance()->attachServer(\EasySwoole\EasySwoole\ServerManager::getInstance()->getSwooleServer());
```

在控制器层响应：

```php
$this->response()->write(\EasySwoole\Template\Render::getInstance()->render('View/custom.html', ['name' => 'easyswoole']));
```
 
## 支持常用的模板引擎
 
下面列举一些常用的模板引擎包方便引入使用:
 
### [smarty/smarty](https://github.com/smarty-php/smarty)
 
Smarty是一个使用PHP写出来的模板引擎,是目前业界最著名的PHP模板引擎之一
 

::: warning 
composer require smarty/smarty=~3.1
:::

 
 
### [league/plates](https://github.com/thephpleague/plates)
 
使用原生PHP语法的非编译型模板引擎，更低的学习成本和更高的自由度
 

::: warning 
composer require league/plates=3.*
:::

 
### [duncan3dc/blade](https://github.com/duncan3dc/blade)
 
Laravel框架使用的模板引擎
 

::: warning 
composer require duncan3dc/blade=^4.5
:::

 
### [topthink/think-template](https://github.com/top-think/think-template)
 
ThinkPHP框架使用的模板引擎
 

::: warning 
 composer require topthink/think-template
:::