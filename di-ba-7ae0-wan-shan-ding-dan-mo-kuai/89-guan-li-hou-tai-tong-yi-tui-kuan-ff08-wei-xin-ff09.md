## 微信支付退款

接下来我们要实现退款的微信支付退款部分。

## 1. 完善退款逻辑

我们需要完善`OrdersController`里面`_refundOrder()`之前预留给微信支付退款的地方：

_app/Admin/Controllers/OrderController.php_

```
.
.
.
    protected function _refundOrder(Order $order)
    {
        switch ($order->payment_method) {
            case 'wechat':
                // 生成退款订单号
                $refundNo = Order::getAvailableRefundNo();
                app('wechat_pay')->refund([
                    'out_trade_no' => $order->no, // 之前的订单流水号
                    'total_fee' => $order->total_amount * 100, //原订单金额，单位分
                    'refund_fee' => $order->total_amount * 100, // 要退款的订单金额，单位分
                    'out_refund_no' => $refundNo, // 退款订单号
                    // 微信支付的退款结果并不是实时返回的，而是通过退款回调来通知，因此这里需要配上退款回调接口地址
                    'notify_url' => 'http://requestbin.fullcontact.com/******' // 由于是开发环境，需要配成 requestbin 地址
                ]);
                // 将订单状态改成退款中
                $order->update([
                    'refund_no' => $refundNo,
                    'refund_status' => Order::REFUND_STATUS_PROCESSING,
                ]);
                break;
            case 'alipay':
                .
                .
                .
            default:
                throw new InternalException('未知订单支付方式：'.$order->payment_method);
                break;
        }
    }
```

## 2. 退款回调控制器

由于微信支付的退款结果并不是实时返回的，而是通过退款回调来通知的，所以接下来我们来实现微信退款回调的接口。

在`PaymentController`里添加一个`wechatRefundNotify()`方法：

_app/Http/Controllers/PaymentController.php_

```
.
.
.
    public function wechatRefundNotify(Request $request)
    {
        // 给微信的失败响应
        $failXml = '<xml><return_code><![CDATA[FAIL]]></return_code><return_msg><![CDATA[FAIL]]></return_msg></xml>';
        $data = app('wechat_pay')->verify(null, true);

        // 没有找到对应的订单，原则上不可能发生，保证代码健壮性
        if(!$order = Order::where('no', $data['out_trade_no'])->first()) {
            return $failXml;
        }

        if ($data['refund_status'] === 'SUCCESS') {
            // 退款成功，将订单退款状态改成退款成功
            $order->update([
                'refund_status' => Order::REFUND_STATUS_SUCCESS,
            ]);
        } else {
            // 退款失败，将具体状态存入 extra 字段，并表退款状态改成失败
            $extra = $order->extra;
            $extra['refund_failed_code'] = $data['refund_status'];
            $order->update([
                'refund_status' => Order::REFUND_STATUS_FAILED,
                'extra' => $extra
            ]);
        }

        return app('wechat_pay')->success();
    }
```

## 3. 退款回调路由

_routes/web.php_

```
Route::post('payment/wechat/refund_notify', 'PaymentController@wechatRefundNotify')->name('payment.wechat.refund_notify');
```

> 和支付回调一样，不能放在有`auth`中间件的路由组中。

最后别忘了把这个路由加到 CSRF 白名单中：

_app/Http/Middleware/VerifyCsrfToken.php_

```
.
.
.
    protected $except = [
        'payment/alipay/notify',
        'payment/wechat/notify',
        'payment/wechat/refund_notify',
    ];
.
.
.
```

## 4. 测试

现在我们来测试一下微信的退款。

先在用户界面创建一笔订单，并用微信支付，然后申请退款，这个过程这里就不再叙述。

> 别忘了从 requestbin 复制回调数据并提交，这样才能让我们系统里记录到对应的微信订单信息。如果忘了怎么操作可以返回到 7.5 节查看。

接着我们访问后台对应的订单详情页，点击退款的`同意`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/9bRX6IFI2K.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/9bRX6IFI2K.png!large)

提示退款成功：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/wQW8QFPvhE.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/wQW8QFPvhE.png!large)

同时微信也收到了退款提醒：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/2FXCXBxnZH.png?imageView2/2/w/1240/h/0 "0")](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/2FXCXBxnZH.png?imageView2/2/w/1240/h/0)

接着我们去看看 requestbin 里面的回调数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/eP5n3ZlA6R.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/eP5n3ZlA6R.png?imageView2/2/w/1240/h/0)

同样的，把`RAW BODY`里的内容复制出来，然后在终端里使用`curl`请求：

```
$ curl -XPOST http://shop.test/payment/wechat/refund_notify -d'{{RAW BODY的内容}}'
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/32s7RxymUE.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/32s7RxymUE.png?imageView2/2/w/1240/h/0)

可以看到返回了成功的字符串，接下来去看看订单的状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/nvJyE7Z3gc.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/nvJyE7Z3gc.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/g3liesbbmY.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/g3liesbbmY.png!large)

状态更新成功。

## 整理退款代码

我们在测试支付和退款的时候为了能够获取到回调请求，在很多地方配置了 requestbin 的地址，现在需要把这些地方改回正确的地址。

_app/Providers/AppServiceProvider.php_

```
.
.
.
    public function register()
    {
        $this->app->singleton('alipay', function () {
            $config               = config('pay.alipay');
            $config['notify_url'] = route('payment.alipay.notify');
            $config['return_url'] = route('payment.alipay.return');
            .
            .
            .
        });

        $this->app->singleton('wechat_pay', function () {
            $config = config('pay.wechat');
            $config['notify_url'] = route('payment.wechat.notify');
            .
            .
            .
        });
    }
```

_app/Admin/Controllers/OrdersController.php_

```
.
.
.
    protected function _refundOrder(Order $order)
    {
        switch ($order->payment_method) {
            case 'wechat':
                $refundNo = Order::getAvailableRefundNo();
                app('wechat_pay')->refund([
                    .
                    .
                    .
                    'notify_url' => route('payment.wechat.refund_notify'),
                ]);
                .
                .
                .
                break;
            case 'alipay':
                .
                .
                .
        }
    }
```

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 同意退款 - 微信支付退款"
```



