## 申请退款

接下来我们要实现的是已支付订单的退款功能。

## 1. 控制器

首先创建一个`ApplyRefundRequest`来校验用户提交的退款申请，要求用户提交退款理由：

```
$ php artisan make:request ApplyRefundRequest
```

退款请求只需要用户提交申请退款理由即可：

_app/Http/Requests/ApplyRefundRequest.php_

```
<?php

namespace App\Http\Requests;

class ApplyRefundRequest extends Request
{
    public function rules()
    {
        return [
            'reason' => 'required',
        ];
    }

    public function attributes()
    {
        return [
            'reason' => '原因',
        ];
    }
}
```

接下来在`OrdersController`中添加`applyRefund()`方法作为提交退款申请的接口：

_app/Http/Controllers/OrdersController.php_

```
use App\Http\Requests\ApplyRefundRequest;
.
.
.
    public function applyRefund(Order $order, ApplyRefundRequest $request)
    {
        // 校验订单是否属于当前用户
        $this->authorize('own', $order);
        // 判断订单是否已付款
        if (!$order->paid_at) {
            throw new InvalidRequestException('该订单未支付，不可退款');
        }
        // 判断订单退款状态是否正确
        if ($order->refund_status !== Order::REFUND_STATUS_PENDING) {
            throw new InvalidRequestException('该订单已经申请过退款，请勿重复申请');
        }
        // 将用户输入的退款理由放到订单的 extra 字段中
        $extra                  = $order->extra ?: [];
        $extra['refund_reason'] = $request->input('reason');
        // 将订单退款状态改为已申请退款
        $order->update([
            'refund_status' => Order::REFUND_STATUS_APPLIED,
            'extra'         => $extra,
        ]);

        return $order;
    }
```

## 2. 路由

然后添加对应的路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::post('orders/{order}/apply_refund', 'OrdersController@applyRefund')->name('orders.apply_refund');
});
```

## 3. 前端模板

接下来我们在订单详情页添加一个申请退款的按钮以及申请退款之后的退款相关信息：

_resources/views/orders/show.blade.php_

```
.
.
.
        <!-- 如果有物流信息则展示 -->
        @if($order->ship_data)
        .
        .
        .
        </div>
        @endif
        <!-- 订单已支付，且退款状态不是未退款时展示退款信息 -->
        @if($order->paid_at && $order->refund_status !== \App\Models\Order::REFUND_STATUS_PENDING)
        <div class="line">
          <div class="line-label">退款状态：</div>
          <div class="line-value">{{ \App\Models\Order::$refundStatusMap[$order->refund_status] }}</div>
        </div>
        <div class="line">
          <div class="line-label">退款理由：</div>
          <div class="line-value">{{ $order->extra['refund_reason'] }}</div>
        </div>
        @endif
.
.
.
        <!-- 如果订单的发货状态为已发货则展示确认收货按钮 -->
        @if($order->ship_status === \App\Models\Order::SHIP_STATUS_DELIVERED)
        .
        .
        .
        @endif
        <!-- 订单已支付，且退款状态是未退款时展示申请退款按钮 -->
        @if($order->paid_at && $order->refund_status === \App\Models\Order::REFUND_STATUS_PENDING)
        <div class="refund-button">
          <button class="btn btn-sm btn-danger" id="btn-apply-refund">申请退款</button>
        </div>
        @endif
.
.
.
@section('scriptsAfterJs')
<script>
  $(document).ready(function () {
  .
  .
  .
    // 退款按钮点击事件
    $('#btn-apply-refund').click(function () {
      swal({
        text: '请输入退款理由',
        content: "input",
      }).then(function (input) {
        // 当用户点击 swal 弹出框上的按钮时触发这个函数
        if(!input) {
          swal('退款理由不可空', '', 'error');
          return;
        }
        // 请求退款接口
        axios.post('{{ route('orders.apply_refund', [$order->id]) }}', {reason: input})
          .then(function () {
            swal('申请退款成功', '', 'success').then(function () {
              // 用户点击弹框上按钮时重新加载页面
              location.reload();
            });
          });
      });
    });

  });
</script>
@endsection
```

调整一下按钮样式，同样是复用之前的样式：

_resources/sass/app.scss_

```
.
.
.
.orders-show-page {
  .
  .
  .
  .payment-buttons, .receive-button, .refund-button {
    margin-top: 10px;
    padding-right: 10px;
  }
}
.
.
.
```

## 4. 测试

接下来我们到订单详情页面看下效果，进入一笔已经支付的订单详情页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/buVAzafLAd.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/buVAzafLAd.png!large)

点击申请退款按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/4yPeHJI9ct.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/4yPeHJI9ct.png!large)

输入退款理由：`测试退款`并提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/RyxjqHvJFn.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/RyxjqHvJFn.png!large)

页面自动刷新，看到`订单状态`那一栏发生了变化：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/EhgdEtloap.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/EhgdEtloap.png!large)

然后再到管理后台的订单列表，看到对应订单退款状态也变为了`已申请退款`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/5iwr1EpEUQ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/5iwr1EpEUQ.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 申请退款"
```



