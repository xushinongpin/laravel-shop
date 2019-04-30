接下来我们需要实现订单的支付功能。

## 1. 控制器

首先创建一个`PaymentController`控制器：

```
$ php artisan make:controller PaymentController
```

在`PaymentController`中添加一个`payByAlipay`方法：

_app/Http/Controllers/PaymentController.php_

```
use App\Models\Order;
use App\Exceptions\InvalidRequestException;
.
.
.
    public function payByAlipay(Order $order, Request $request)
    {
        // 判断订单是否属于当前用户
        $this->authorize('own', $order);
        // 订单已支付或者已关闭
        if ($order->paid_at || $order->closed) {
            throw new InvalidRequestException('订单状态不正确');
        }

        // 调用支付宝的网页支付
        return app('alipay')->web([
            'out_trade_no' => $order->no, // 订单编号，需保证在商户端不重复
            'total_amount' => $order->total_amount, // 订单金额，单位元，支持小数点后两位
            'subject'      => '支付 Laravel Shop 的订单：'.$order->no, // 订单标题
        ]);
    }
```

## 2. 路由

接下来添加对应的路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
            Route::get('payment/{order}/alipay', 'PaymentController@payByAlipay')->name('payment.alipay');
});
```

## 3. 添加支付入口

接下来我们需要在未支付的订单页面加入支付宝支付的按钮：

_resources/views/orders/show.blade.php_

```
.
.
.
<div>
  <span>订单状态：</span>
  .
  .
  .
</div>
<!-- 支付按钮开始 -->
@if(!$order->paid_at && !$order->closed)
<div class="payment-buttons">
  <a class="btn btn-primary btn-sm" href="{{ route('payment.alipay', ['order' => $order->id]) }}">支付宝支付</a>
</div>
@endif
<!-- 支付按钮结束 -->
.
.
.
```

然后配置一下按钮的样式：

_resources/sass/app.scss_

```
.
.
.
.orders-show-page {
  .
  .
  .
  .payment-buttons {
    margin-top: 10px;
    padding-right: 10px;
  }
}
```

现在访问一下没有被关闭的订单详情页看看效果：：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/KKZ5a0RSjI.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/KKZ5a0RSjI.png?imageView2/2/w/1240/h/0)

点击`支付宝支付`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/xTGNckV9A8.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/xTGNckV9A8.png?imageView2/2/w/1240/h/0)

可以看到对应的标题和金额。继续后面的登录和支付流程：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/23/5320/axoXW5KpPk.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/23/5320/axoXW5KpPk.png?imageView2/2/w/1240/h/0)

支付成功之后发现页面一直停留在支付宝的页面没有跳转，这是因为我们没有配置支付的前端回调，我们之后会说到。

现在再次访问刚刚的订单页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/9i3NsEZdN8.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/9i3NsEZdN8.png?imageView2/2/w/1240/h/0)

发现状态还是未支付。这是因为我们现在还没写更新状态的逻辑，接下来我们就要来实现这块。

## 4. 支付回调

支付宝的支付回调分为**前端回调**和**服务器回调**。

* **前端回调**是指当用户支付成功之后支付宝会让用户浏览器跳转回项目页面并带上支付成功的参数，也就是说前端回调依赖于用户浏览器，如果用户在跳转之前关闭浏览器，将无法收到前端回调。

* **服务器回调**  
  是指支付成功之后支付宝的服务器会用订单相关数据作为参数请求项目的接口，不依赖用户浏览器。

因此我们判断支付是否成功要以服务器端回调为准。

我们需要在`PaymentController`中新增两个方法，分别用于处理前端和服务器端回调。

_app/Http/Controllers/PaymentController.php_

```
.
.
.
    // 前端回调页面
    public function alipayReturn()
    {
        // 校验提交的参数是否合法
        $data = app('alipay')->verify();
        dd($data);
    }

    // 服务器端回调
    public function alipayNotify()
    {
        $data = app('alipay')->verify();
        \Log::debug('Alipay notify', $data->all());
    }
```

代码解析：

* `app('alipay')-`
  `>`
  `verify()`
  用于校验提交的参数是否合法，支付宝的前端跳转会带有数据签名，通过校验数据签名可以判断参数是否被恶意用户篡改。同时该方法还会返回解析后的参数。
* `dd($data);`
  输出解析后的数据，我们要先看看会返回什么再决定如何处理。
* `\Log::debug('Alipay notify', $data-`
  `>`
  `all());`
  由于服务器端的请求我们无法看到返回值，使用
  `dd`
  就不行了，所以需要通过日志的方式来保存。

接下来将这两个方法注册到路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::get('payment/alipay/return', 'PaymentController@alipayReturn')->name('payment.alipay.return');
});
Route::post('payment/alipay/notify', 'PaymentController@alipayNotify')->name('payment.alipay.notify');
```

> 服务器端回调的路由不能放到带有`auth`中间件的路由组中，因为支付宝的服务器请求不会带有认证信息。

接下来我们把这两个回调地址配置到支付宝的支付实例里：

_app/Providers/AppServiceProvider.php_

```
.
.
.
        $this->app->singleton('alipay', function () {
            $config               = config('pay.alipay');
            $config['notify_url'] = route('payment.alipay.notify');
            $config['return_url'] = route('payment.alipay.return');
            .
            .
            .
        });
.
.
.
```

`notify_url`代表服务器端回调地址，`return_url`代表前端回调地址。

> 注意：回调地址必须是完整的带有域名的 URL，不可以是相对路径。使用`route()`函数生成的 URL 默认就是带有域名的完整地址。

接下来我们再一次测试支付流程。

> 需要创建一个新的订单来支付，因为刚刚那个订单在支付宝的系统里已经认为是支付成功了。

支付完成之后看到支付宝页面准备跳转回商户页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/r2Fz2Kek4Q.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/r2Fz2Kek4Q.png?imageView2/2/w/1240/h/0)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uwe3zmwZ50.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uwe3zmwZ50.png!large)

可以看到已经跳转到我们的前端回调页面并且通过`dd()`函数将回调的数据打印了出来。图中框起来的部分就是我们支付的这笔订单的流水号。

但是我们并没有收到服务器端回调，这是因为我们的服务器运行在本地，`https://shop.test`这个域名也只是对我们本地生效，支付宝服务器无法请求到我们的服务器端回调地址。

我们可以通过`requestbin`服务来帮我们捕获服务器端回调的数据。

> `requestbin`是一个免费开源的网站，任何人都可以在上面申请一个专属的 URL（通常有效期 48 小时），对这个 URL 的任何类型的请求都会被记录下来，URL 的创建者可以看到请求的具体信息，包含请求时间、请求头、请求具体数据等。

访问[https://requestbin.fullcontact.com/](https://requestbin.fullcontact.com/)，点击中间那个绿按钮`Create a RequestBin`：

> 如果上面的网站因为网络问题，不能正常使用，可以使用[https://requestbin.leo108.com/](https://requestbin.leo108.com/)作为替代。这是我专为此课程搭的一个 RequestBin 国内镜像服务。

[![](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/PaEUkXpl54.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/PaEUkXpl54.png?imageView2/2/w/1240/h/0)

系统就会给你分配一个 URL：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/v3JFpVJzdk.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/v3JFpVJzdk.png?imageView2/2/w/1240/h/0)

我们把这个 URL 复制下来，放到我们之前放服务器端回调地址的参数上：

_app/Providers/AppServiceProvider.php_

```
.
.
.
        $this->app->singleton('alipay', function () {
            $config               = config('pay.alipay');
            $config['notify_url'] = 'http://requestbin.fullcontact.com/128xjcv1';
            $config['return_url'] = route('payment.alipay.return');
            .
            .
            .
        });
.
.
.
```

再创建一笔订单，走一次支付流程，等到前端回调完成之后，我们点击 requestbin 页面右上角文本框左侧的那个圆点（颜色可能不一样），会跳转到请求日志页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/egHoak6hvZ.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/egHoak6hvZ.png?imageView2/2/w/1240/h/0)

可以看到里面已经有请求数据了：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/7euoHYPPsB.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/7euoHYPPsB.png?imageView2/2/w/1240/h/0)

接下来我们试着把这个请求数据提交到我们本地的回调 URL，把页面上`RAW BODY`（原始请求数据）下方的数据复制下来：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/nliScppmEX.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/nliScppmEX.png?imageView2/2/w/1240/h/0)

然后在终端使用`curl`来请求我们的服务器端回调 URL：

```
$ curl -XPOST http://shop.test/payment/alipay/notify -d'复制 raw body 内容'
```

> 注意请求内容的两端都要加上单引号

[![](https://iocaffcdn.phphub.org/uploads/images/201904/20/5320/k7aFzyY321.png!large "订单的支付宝支付")](https://iocaffcdn.phphub.org/uploads/images/201904/20/5320/k7aFzyY321.png!large)

发现返回的结果不对，居然返回了一个 Html 页面，标题是`页面会话已超时`。在 Laravel 里`页面会话已超时`通常代表 Post 请求缺少 CSRF Token 或者 CSRF Token 过期。由于我们这个 URL 是给支付宝服务器调用的，肯定不会有 CSRF Token，所以需要把这个 URL 加到 CSRF 白名单里：

_app/Http/Middleware/VerifyCsrfToken.php_

```
.
.
.
    protected $except = [
        'payment/alipay/notify',
    ];
.
.
.
```

如果访问的 URL 能够匹配上`$except`里任意一项，Laravel 就不会去检查 CSRF Token。

在终端里按`↑`键可以直接回到上一条命令，再次回车：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/Q1iX0OinYA.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/Q1iX0OinYA.png?imageView2/2/w/1240/h/0)

可以看到这回没有返回任何错误信息了，我们去看一下日志里保存了什么数据。

> 一定要用`less`命令查看日志文件，而不是`vim`或者其他编辑器。通常来说日志文件会特别大，特别是线上的服务器，如果用`vim`或者其他编辑器打开，会把整个日志文件加载到内存中，有可能立马将服务器内存撑爆导致系统故障。`less`命令则不会，可以理解为按需加载文件内容到内存中。

```
$ less storage/logs/laravel-{当前日期}.log
```

默认情况是显示日志的开头几行，按`shift + g`可以跳转到日志文件的末尾：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/zWrduTvrNx.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/zWrduTvrNx.png?imageView2/2/w/1240/h/0)

可以看到我们的打日志的代码成功运行了，记录下解析后的支付宝服务器端回调数据。

现在我们已经知道支付宝的回调有哪些数据了，接下来我们要完善一下两个回调接口：

_app/Http/Controllers/PaymentController.php_

```
use Carbon\Carbon;
.
.
.
    public function alipayReturn()
    {
        try {
            app('alipay')->verify();
        } catch (\Exception $e) {
            return view('pages.error', ['msg' => '数据不正确']);
        }

        return view('pages.success', ['msg' => '付款成功']);
    }

    public function alipayNotify()
    {
        // 校验输入参数
        $data  = app('alipay')->verify();
        // 如果订单状态不是成功或者结束，则不走后续的逻辑
        // 所有交易状态：https://docs.open.alipay.com/59/103672
        if(!in_array($data->trade_status, ['TRADE_SUCCESS', 'TRADE_FINISHED'])) {
            return app('alipay')->success();
        }
        // $data->out_trade_no 拿到订单流水号，并在数据库中查询
        $order = Order::where('no', $data->out_trade_no)->first();
        // 正常来说不太可能出现支付了一笔不存在的订单，这个判断只是加强系统健壮性。
        if (!$order) {
            return 'fail';
        }
        // 如果这笔订单的状态已经是已支付
        if ($order->paid_at) {
            // 返回数据给支付宝
            return app('alipay')->success();
        }

        $order->update([
            'paid_at'        => Carbon::now(), // 支付时间
            'payment_method' => 'alipay', // 支付方式
            'payment_no'     => $data->trade_no, // 支付宝订单号
        ]);

        return app('alipay')->success();
    }
```

其中`app('alipay')->success()`返回数据给支付宝，支付宝得到这个返回之后就认为我们已经处理好这笔订单，不会再发生这笔订单的回调了。如果我们返回其他数据给支付宝，支付宝就会每隔一段时间就发送一次服务器端回调，直到我们返回了正确的数据为准。

在前端回调里我们要展示付款成功信息给用户，因此需要一个对应的前端模板：

```
$ touch resources/views/pages/success.blade.php
```

_resources/views/pages/success.blade.php_

```
@extends('layouts.app')
@section('title', '操作成功')

@section('content')
  <div class="card">
    <div class="card-header">操作成功</div>
    <div class="card-body text-center">
      <h1>{{ $msg }}</h1>
      <a class="btn btn-primary" href="{{ route('root') }}">返回首页</a>
    </div>
  </div>
@endsection
```

接下来我们再在终端中发起回调请求：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/EuZCxa3WsI.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/EuZCxa3WsI.png?imageView2/2/w/1240/h/0)

可以看到我们的接口返回了`success`。

再到数据库中看看订单的状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/QZiGgr254d.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/QZiGgr254d.png?imageView2/2/w/1240/h/0)

`paid_at`、`payment_method`和`payment_no`也都有数据了。

再访问我们的订单列表页面：[http://shop.test/orders](http://shop.test/orders)

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/Z4KZ4mUecu.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/Z4KZ4mUecu.png?imageView2/2/w/1240/h/0)

页面上显示的状态也变成了`已支付`。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "订单支付"
```



