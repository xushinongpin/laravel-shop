[![](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/cF69b4gcAl.jpeg?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/cF69b4gcAl.jpeg?imageView2/2/w/1240/h/0)

## 支付

上一章我们已经实现了订单的创建，接下来我们要实现支付功能，这也是电商系统的核心功能。

## 1. 引入支付库

`yansongda/pay`这个库封装了支付宝和微信支付的接口，通过这个库我们就不需要去关注不同支付平台的接口差异，使用相同的方法、参数来完成支付功能，节省开发时间。

首先通过`composer`引入这个包：

```
$ composer require yansongda/pay
```

## 2. 配置参数

我们创建一个新的配置文件来保存支付所需的参数：

```
$ touch config/pay.php
```

_config/pay.php_

```
<?php

return [
    'alipay' => [
        'app_id'         => '',
        'ali_public_key' => '',
        'private_key'    => '',
        'log'            => [
            'file' => storage_path('logs/alipay.log'),
        ],
    ],

    'wechat' => [
        'app_id'      => '',
        'mch_id'      => '',
        'key'         => '',
        'cert_client' => '',
        'cert_key'    => '',
        'log'         => [
            'file' => storage_path('logs/wechat_pay.log'),
        ],
    ],
];
```

现在这些参数先留空，后面的章节我们再填入具体的值。

## 3. 容器

容器是现代 PHP 开发的一个重要概念，Laravel 就是在容器的基础上构建的。我们将支付操作类实例注入到容器中，在以后的代码里就可以直接通过`app('alipay')`来取得对应的实例，而不需要每次都重新创建。

我们通常在`AppServiceProvider`的`register()`方法中往容器中注入实例：

_app/Providers/AppServiceProvider.php_

```
use Monolog\Logger;
use Yansongda\Pay\Pay;
.
.
.
    public function register()
    {
        // 往服务容器中注入一个名为 alipay 的单例对象
        $this->app->singleton('alipay', function () {
            $config = config('pay.alipay');
            // 判断当前项目运行环境是否为线上环境
            if (app()->environment() !== 'production') {
                $config['mode']         = 'dev';
                $config['log']['level'] = Logger::DEBUG;
            } else {
                $config['log']['level'] = Logger::WARNING;
            }
            // 调用 Yansongda\Pay 来创建一个支付宝支付对象
            return Pay::alipay($config);
        });

        $this->app->singleton('wechat_pay', function () {
            $config = config('pay.wechat');
            if (app()->environment() !== 'production') {
                $config['log']['level'] = Logger::DEBUG;
            } else {
                $config['log']['level'] = Logger::WARNING;
            }
            // 调用 Yansongda\Pay 来创建一个微信支付对象
            return Pay::wechat($config);
        });
    }
```

代码解析：

* `$this-`
  `>`
  `app-`
  `>`
  `singleton()`
  往服务容器中注入一个单例对象，第一次从容器中取对象时会调用回调函数来生成对应的对象并保存到容器中，之后再去取的时候直接将容器中的对象返回。
* `app()-`
  `>`
  `environment()`
  获取当前运行的环境，线上环境会返回
  `production`
  。对于支付宝，如果项目运行环境不是线上环境，则启用开发模式，并且将日志级别设置为
  `DEBUG`
  。由于微信支付没有开发模式，所以仅仅将日志级别设置为
  `DEBUG`
  。

## 4. 测试

接下来我们来测试一下刚刚注入到容器中的实例，进入 tinker：

```
$ php artisan tinker
```

然后分别输入`app('alipay')`和`app('wechat_pay')`

```
>>> app('alipay')
>>> app('wechat_pay')
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/5fLw9s31VH.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/5fLw9s31VH.png?imageView2/2/w/1240/h/0)

可以看到返回了`Yansongda\Pay\Gateways\Alipay`和`Yansongda\Pay\Gateways\Wechat`类型的对象，说明注入成功。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "引入支付库"
```



