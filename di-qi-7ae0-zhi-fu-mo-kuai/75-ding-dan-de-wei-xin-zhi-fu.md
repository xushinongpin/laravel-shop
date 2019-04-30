## 微信支付

接下来我们来实现微信支付的入口。

## 1. 控制器

接下来我们来实现微信支付的入口，与支付宝类似，在`PaymentController`中新增一个`payByWechat()`方法：

_app/Http/Controllers/PaymentController.php_

```
.
.
.
    public function payByWechat(Order $order, Request $request) {
        // 校验权限
        $this->authorize('own', $order);
        // 校验订单状态
        if ($order->paid_at || $order->closed) {
            throw new InvalidRequestException('订单状态不正确');
        }
        // scan 方法为拉起微信扫码支付
        return app('wechat_pay')->scan([
            'out_trade_no' => $order->no,  // 商户订单流水号，与支付宝 out_trade_no 一样
            'total_fee' => $order->total_amount * 100, // 与支付宝不同，微信支付的金额单位是分。
            'body'      => '支付 Laravel Shop 的订单：'.$order->no, // 订单描述
        ]);
    }
    .
    .
    .
```

然后是回调接口，微信的扫码支付没有前端回调只有服务器端回调：

_app/Http/Controllers/PaymentController.php_

```
.
.
.
    public function wechatNotify()
    {
        // 校验回调参数是否正确
        $data  = app('wechat_pay')->verify();
        // 找到对应的订单
        $order = Order::where('no', $data->out_trade_no)->first();
        // 订单不存在则告知微信支付
        if (!$order) {
            return 'fail';
        }
        // 订单已支付
        if ($order->paid_at) {
            // 告知微信支付此订单已处理
            return app('wechat_pay')->success();
        }

        // 将订单标记为已支付
        $order->update([
            'paid_at'        => Carbon::now(),
            'payment_method' => 'wechat',
            'payment_no'     => $data->transaction_id,
        ]);

        return app('wechat_pay')->success();
    }
```

## 2. 路由

接下来配置路由：

_routes/web.php_

```
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::get('payment/{order}/wechat', 'PaymentController@payByWechat')->name('payment.wechat');
});
Route::post('payment/wechat/notify', 'PaymentController@wechatNotify')->name('payment.wechat.notify');
```

## 3. 配置回调地址

和支付宝一样，服务器端回调我们先用`requestbin`来捕获：

_app/Providers/AppServiceProvider.php_

```
.
.
.
        $this->app->singleton('wechat_pay', function () {
            $config = config('pay.wechat');
            $config['notify_url'] = 'http://requestbin.fullcontact.com/[替换成你自己的url]';
            .
            .
            .
        });
```

别忘了把微信的服务器端回调地址加入 CSRF 校验白名单：

_app/Http/Middleware/VerifyCsrfToken.php_

```
   protected $except = [
        'payment/alipay/notify',
        'payment/wechat/notify',
    ];
```

## 4. 前端添加支付入口

接下来我们在支付宝支付按钮后面加上微信支付按钮：

_resources/views/order/show.blade.php_

```
.
.
.
        <div class="payment-buttons">
          <a class="btn btn-primary btn-sm" href="{{ route('payment.alipay', ['order' => $order->id]) }}">支付宝支付</a>
          <a class="btn btn-success btn-sm" href="{{ route('payment.wechat', ['order' => $order->id]) }}">微信支付</a>
        </div>
.
.
.
```

## 5. 支付测试

由于微信支付是在真实的支付环境中测试的，因此我们要把商品金额改成一分钱，登录到管理后台，将一个商品的一个 SKU 的价格改成`0.01`：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/xeyYinmVhx.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/xeyYinmVhx.png?imageView2/2/w/1240/h/0)

然后将这个 SKU 加入购物车并下单。

可以看到微信支付的按钮了：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/RaYhxa7m2z.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/RaYhxa7m2z.png?imageView2/2/w/1240/h/0)

点击`微信支付`，发现输出了一个 JSON 字符串：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/Y7QSelcwv8.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/Y7QSelcwv8.png?imageView2/2/w/1240/h/0)

根据微信支付的文档，我们需要把 JSON 字符串中的`code_url`转成二维码展示给用户扫描，我们这里先用[https://cli.im/](https://cli.im/)这个网站手动将其转为二维码：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/qvaQ5hxTqo.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/qvaQ5hxTqo.png?imageView2/2/w/1240/h/0)

将`code_url`复制下来粘贴到左侧的框中，注意要把转义字符去掉。然后点击`生成二维码`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/EaEEYkurju.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/EaEEYkurju.png?imageView2/2/w/1240/h/0)

拿起微信扫描右侧的二维码，可以看到拉起了微信支付页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/dksbIm4kY3.png?imageView2/2/w/1240/h/0 "0")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/dksbIm4kY3.png?imageView2/2/w/1240/h/0)

输入密码完成支付：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/1WtaGIMO9L.jpeg?imageView2/2/w/1240/h/0 "0")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/1WtaGIMO9L.jpeg?imageView2/2/w/1240/h/0)

接下来我们去看看`requestbin`里捕获到的服务器端回调数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/1vpPiCrmxW.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/1vpPiCrmxW.png?imageView2/2/w/1240/h/0)

可以看到是一个 XML 格式的数据，直接将`RAW BODY`的数据复制下来。

进入终端，使用`curl`请求我们的服务器端回调接口：

```
$ curl -XPOST http://shop.test/payment/wechat/notify -d'XML请求包'
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/ht9Gi06xqz.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/ht9Gi06xqz.png?imageView2/2/w/1240/h/0)

可以看到我们的接口也返回了一个代表成功的 XML 格式数据。

检查一下数据库：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/oz2lTMIQpU.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/oz2lTMIQpU.png?imageView2/2/w/1240/h/0)

订单状态正确。

## 6. 生成二维码

我们刚刚在点击`微信支付`之后页面显示的是 JSON 字符串，我们不可能让用户自己跑到别的网站去转成二维码，因此我们需要通过代码将`code_url`转成二维码并输出。

首先通过`composer`引入`endroid/qr-code`这个库：

```
$ composer require endroid/qr-code
```

接下来使用这个库将支付 URL 转成二维码：

_app/Http/Controllers/PaymentController.php_

```
use Endroid\QrCode\QrCode;
.
.
.
    public function payByWechat(Order $order, Request $request)
    {
        $this->authorize('own', $order);
        if ($order->paid_at || $order->closed) {
            throw new InvalidRequestException('订单状态不正确');
        }

        // 之前是直接返回，现在把返回值放到一个变量里
        $wechatOrder = app('wechat_pay')->scan([
            'out_trade_no' => $order->no,
            'total_fee'    => $order->total_amount * 100,
            'body'         => '支付 Laravel Shop 的订单：'.$order->no,
        ]);
        // 把要转换的字符串作为 QrCode 的构造函数参数
        $qrCode = new QrCode($wechatOrder->code_url);

        // 将生成的二维码图片数据以字符串形式输出，并带上相应的响应类型
        return response($qrCode->writeString(), 200, ['Content-Type' => $qrCode->getContentType()]);
    }
    .
    .
    .
```

[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/VS9ZH1Xljd.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/VS9ZH1Xljd.png?imageView2/2/w/1240/h/0)

但是直接跳转到一个二维码图片的页面体验也不是很好，接下来我们把这个改造成弹框：

_resources/views/orders/show.blade.php_

```
.
.
.
<!-- 把之前的微信支付按钮换成这个 -->
<button class="btn btn-sm btn-success" id='btn-wechat'>微信支付</button>
.
.
.
@section('scriptsAfterJs')
<script>
  $(document).ready(function() {
    // 微信支付按钮事件
    $('#btn-wechat').click(function() {
      swal({
        // content 参数可以是一个 DOM 元素，这里我们用 jQuery 动态生成一个 img 标签，并通过 [0] 的方式获取到 DOM 元素
        content: $('<img src="{{ route('payment.wechat', ['order' => $order->id]) }}" />')[0],
        // buttons 参数可以设置按钮显示的文案
        buttons: ['关闭', '已完成付款'],
      })
      .then(function(result) {
      // 如果用户点击了 已完成付款 按钮，则重新加载页面
        if (result) {
          location.reload();
        }
      })
    });
  });
</script>
@endsection
```

刷新页面，再点击`微信支付`按钮看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/XGg2m1DhcT.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/XGg2m1DhcT.png?imageView2/2/w/1240/h/0)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "微信支付"
```



