## 确认收货

上一节我们完成了后台的发货操作，这一节我们要来实现用户端确认收货的功能。

## 1. 控制器

首先在`OrdersController`中新增`received()`方法作为确认收货的接口：

_app/Http/Controllers/OrdersController.php_

```
use App\Exceptions\InvalidRequestException;
.
.
.
    public function received(Order $order, Request $request)
    {
        // 校验权限
        $this->authorize('own', $order);

        // 判断订单的发货状态是否为已发货
        if ($order->ship_status !== Order::SHIP_STATUS_DELIVERED) {
            throw new InvalidRequestException('发货状态不正确');
        }

        // 更新发货状态为已收到
        $order->update(['ship_status' => Order::SHIP_STATUS_RECEIVED]);

        // 返回原页面
        return redirect()->back();
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
    Route::post('orders/{order}/received', 'OrdersController@received')->name('orders.received');
});
```

## 3. 前端模板

接下来我们要调整一下用户界面的订单详情页模板，展示物流状态，并且当订单中有物流信息时将其展示出来：

_resources/views/orders/show.blade.php_

```
.
.
.
        <div class="line"><div class="line-label">订单编号：</div><div class="line-value">{{ $order->no }}</div></div>
        <!-- 输出物流状态 -->
        <div class="line">
          <div class="line-label">物流状态：</div>
          <div class="line-value">{{ \App\Models\Order::$shipStatusMap[$order->ship_status] }}</div>
        </div>
        <!-- 如果有物流信息则展示 -->
        @if($order->ship_data)
        <div class="line">
          <div class="line-label">物流信息：</div>
          <div class="line-value">{{ $order->ship_data['express_company'] }} {{ $order->ship_data['express_no'] }}</div>
        </div>
        @endif
.
.
.
        @if(!$order->paid_at && !$order->closed)
        .
        .
        .
        @endif
        <!-- 如果订单的发货状态为已发货则展示确认收货按钮 -->
        @if($order->ship_status === \App\Models\Order::SHIP_STATUS_DELIVERED)
        <div class="receive-button">
          <form method="post" action="{{ route('orders.received', [$order->id]) }}">
            <!-- csrf token 不能忘 -->
            {{ csrf_field() }}
            <button type="submit" class="btn btn-sm btn-success">确认收货</button>
          </form>
        </div>
        @endif
.
.
.
```

## 4. 测试

接下来我们测试一下，访问[http://shop.test/orders](http://shop.test/orders)进入用户界面的订单列表，进入刚刚我们在后台操作已发货的订单详情页，可以看到在最下方展示了物流信息，并且有一个`确认收货`的按钮，点击这个按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/RRw9N4GYba.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/RRw9N4GYba.png!large)

页面自动刷新，物流状态变更为`已收货`。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qAZfRu6In9.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qAZfRu6In9.png!large)

再到后台看对应的订单详情页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/k0cMlgIWU0.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/k0cMlgIWU0.png!large)

发货状态显示为`已收货`。

## 5. 体验优化

确认收货的按钮样式不太好，需要调整一下，可以直接复用支付按钮的样式：

_resources/sass/app.scss_

```
.
.
.
.orders-show-page {
  .
  .
  .
  .payment-buttons, .receive-button {
    margin-top: 10px;
    padding-right: 10px;
  }
}
.
.
.
```

现在看一下效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/mobeVdajZv.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/mobeVdajZv.png!large)

另外一个比较严重的问题是在用户端确认收货的时候没有二次确认就直接提交了，这样用户很容易因为误操作就够确认收货了，因此我们需要在用户点击确认收货时候弹出一个框让用户确认。

_resources/views/orders/show.blade.php_

```
.
.
.
@if($order->ship_status === \App\Models\Order::SHIP_STATUS_DELIVERED)
<div class="receive-button">
  <!-- 将原本的表单替换成下面这个按钮 -->
  <button type="button" id="btn-receive" class="btn btn-sm btn-success">确认收货</button>
</div>
@endif
.
.
.
@section('scriptsAfterJs')
<script>
  $(document).ready(function() {
    .
    .
    .
    // 确认收货按钮点击事件
    $('#btn-receive').click(function() {
      // 弹出确认框
      swal({
        title: "确认已经收到商品？",
        icon: "warning",
        dangerMode: true,
        buttons: ['取消', '确认收到'],
      })
      .then(function(ret) {
        // 如果点击取消按钮则不做任何操作
        if (!ret) {
          return;
        }
        // ajax 提交确认操作
        axios.post('{{ route('orders.received', [$order->id]) }}')
          .then(function () {
            // 刷新页面
            location.reload();
          })
      });
    });

  });
</script>
@endsection
```

由于我们把确认收货的操作从表单提交改成了 AJAX 请求，因此控制器中的返回值需要修改一下：

_app/Http/Controllers/OrdersController.php_

```
.
.
.
    public function received(Order $order, Request $request)
    {
        .
        .
        .
        // 返回订单信息
        return $order;
    }
```

到页面里看下效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/FJNyBzQP8H.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/FJNyBzQP8H.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "确认收货"
```



