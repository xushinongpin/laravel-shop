## 优雅地处理异常

上一节我们在实现商品详情页的时候，在处理非正常流程时使用了`throw new Exception`抛出异常来终止流程：

```
if (!$product->on_sale) {
    throw new \Exception('商品未上架');
}
```

大家可以尝试访问一个被下架的商品来触发这个异常，在开发环境会看到类似这样的界面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/AMS2DvR1IA.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/AMS2DvR1IA.png!large)

而当线上环境的用户触发了这个异常时就会看到：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/p4KfuepEb0.png!large "优雅地处理异常")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/p4KfuepEb0.png!large)

这样的提示对用户很不友好。

本章节将要介绍在 Laravel 项目中应该如何正确地、优雅地处理异常。

## 异常

异常指的是在程序运行过程中发生的异常事件，通常是由外部问题所导致的。

异常处理是程序开发中经常遇到的任务，如何优雅地处理异常，从一定程度上反映了你的程序是否足够严谨。

在本次的项目开发中，我们将异常大致分为**用户异常**和**系统异常**，接下来我们将分别对其讲解和代码实现。

### 1. 用户错误行为触发的异常

比如访问一个被下架的商品时触发的异常，对于此类异常我们需要把触发异常的原因告知用户。

我们把这类异常命名为`InvalidRequestException`，可以通过`make:exception`命令来创建：

```
$ php artisan make:exception InvalidRequestException
```

新创建的异常文件保存在`app/Exceptions/`目录下：

_app/Exceptions/InvalidRequestException.php_

```
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Http\Request;

class InvalidRequestException extends Exception
{
    public function __construct(string $message = "", int $code = 400)
    {
        parent::__construct($message, $code);
    }

    public function render(Request $request)
    {
        if ($request->expectsJson()) {
            // json() 方法第二个参数就是 Http 返回码
            return response()->json(['msg' => $this->message], $this->code);
        }

        return view('pages.error', ['msg' => $this->message]);
    }
}
```

Laravel 5.5 之后支持在异常类中定义`render()`方法，该异常被触发时系统会调用`render()`方法来输出，我们在`render()`里判断如果是 AJAX 请求则返回 JSON 格式的数据，否则就返回一个错误页面。

现在来创建这个错误页面：

```
$ touch resources/views/pages/error.blade.php
```

_resources/views/pages/error.blade.php_

```
@extends('layouts.app')
@section('title', '错误')

@section('content')
<div class="card">
    <div class="card-header">错误</div>
    <div class="card-body text-center">
        <h1>{{ $msg }}</h1>
        <a class="btn btn-primary" href="{{ route('root') }}">返回首页</a>
    </div>
</div>
@endsection
```

当异常触发时 Laravel 默认会把异常的信息和调用栈打印到日志里，比如：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/hHHU4lNHYr.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/hHHU4lNHYr.png!large)

而此类异常并不是因为我们系统本身的问题导致的，不会影响我们系统的运行，如果大量此类日志打印到日志文件里反而会影响我们去分析真正有问题的异常，因此需要屏蔽这个行为。

Laravel 内置了屏蔽指定异常写日志的解决方案：

_app/Exceptions/Handler.php_

```
.
.
.
    protected $dontReport = [
        InvalidRequestException::class,
    ];
.
.
.
```

当一个异常被触发时，Laravel 会去检查这个异常的类型是否在`$dontReport`属性中定义了，如果有则不会打印到日志文件中。

### 2. 系统内部异常

比如连接数据库失败，对于此类异常我们需要有限度地告知用户发生了什么，但又不能把所有信息都暴露给用户（比如连接数据库失败的信息里会包含数据库地址和账号密码），因此我们需要传入两条信息，一条是给用户看的，另一条是打印到日志中给开发人员看的。

新建一个`InternalException`类：

```
$ php artisan make:exception InternalException
```

_app/Exceptions/InternalException.php_

```
<?php

namespace App\Exceptions;

use Exception;
use Illuminate\Http\Request;

class InternalException extends Exception
{
    protected $msgForUser;

    public function __construct(string $message, string $msgForUser = '系统内部错误', int $code = 500)
    {
        parent::__construct($message, $code);
        $this->msgForUser = $msgForUser;
    }

    public function render(Request $request)
    {
        if ($request->expectsJson()) {
            return response()->json(['msg' => $this->msgForUser], $this->code);
        }

        return view('pages.error', ['msg' => $this->msgForUser]);
    }
}
```

这个异常的构造函数第一个参数就是原本应该有的异常信息比如连接数据库失败，第二个参数是展示给用户的信息，通常来说只需要告诉用户`系统内部错误`即可，因为不管是连接 Mysql 失败还是连接 Redis 失败对用户来说都是一样的，就是系统不可用，用户也不可能根据这个信息来解决什么问题。

## 应用到代码中

接下来我们要把之前判断下架商品的异常替换成我们刚刚定义的异常。

_app/Http/Controllers/ProductsController.php_

```
use App\Exceptions\InvalidRequestException;
.
.
.
    public function show(Product $product, Request $request)
    {
        if (!$product->on_sale) {
            throw new InvalidRequestException('商品未上架');
        }
        .
        .
        .
    }
```

接下来我们看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/ebLRE0K1Gv.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/ebLRE0K1Gv.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "异常处理"
```



