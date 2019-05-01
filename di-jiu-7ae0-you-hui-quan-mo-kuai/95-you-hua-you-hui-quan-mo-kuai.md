## 优化优惠券模块

上一节我们完成了使用优惠券来下单的基本功能，接下来我们要完善一些遗漏的功能。

## 一张优惠券一个用户只能使用一次

通常来说一张优惠券对每一个用户来说只能使用一次，这里我们对『使用』的定义是：有关联了此优惠券的**未付款且未关闭订单**或者**已付款且未退款成功订单**，根据这个规则我们来完善一下`CouponCode`模型中的`checkAvailable()`方法：

_app/Models/CouponCode.php_

```
.
.
.
    // 添加了一个 $user 参数
    public function checkAvailable(User $user, $orderAmount = null)
    {
        .
        .
        .
        $used = Order::where('user_id', $user->id)
            ->where('coupon_code_id', $this->id)
            ->where(function($query) {
                $query->where(function($query) {
                    $query->whereNull('paid_at')
                        ->where('closed', false);
                })->orWhere(function($query) {
                    $query->whereNotNull('paid_at')
                        ->where('refund_status', '!=', Order::REFUND_STATUS_SUCCESS);
                });
            })
            ->exists();
        if ($used) {
            throw new CouponCodeUnavailableException('你已经使用过这张优惠券了');
        }
    }
```

这里的查询构造器最终生成的 SQL 类似：

```
select * from orders where user_id = xx and coupon_code_id = xx
  and (
    ( paid_at is null and closed = 0 )
    or ( paid_at is not null and refund_status != 'success' )
  )
```

我们代码中使用`where(function($query) {})`嵌套是用来生成的 SQL 里的括号，保证不会因为`or`关键字导致我们的查出来的结果与期望不符。

由于我们修改了参数，因此需要修改调用这方法的地方：

_app/Http/Controllers/CouponCodesController.php_

```
use Illuminate\Http\Request;
.
.
.
    public function show($code, Request $request)
    {
        .
        .
        .
        $record->checkAvailable($request->user());

        return $record;
    }
```

_app/Services/OrderService.php_

```
.
.
.
    public function store(User $user, UserAddress $address, $remark, $items, CouponCode $coupon = null)
    {
        if ($coupon) {
            $coupon->checkAvailable($user);
        }
        $order = \DB::transaction(function () use ($user, $address, $remark, $items, $coupon) {
        .
        .
        .
            if ($coupon) {
                $coupon->checkAvailable($user, $totalAmount);
                .
                .
                .
            }
        }
        .
        .
        .
    }
```

现在来测试一下：

在购物车中添加一件商品，然后再次使用我们之前用过的那个 25% 优惠券试试看：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/y2kQEF3C4k.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/y2kQEF3C4k.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0PN6gUuYs9.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0PN6gUuYs9.png!large)

符合预期。

## 订单展示优惠信息

现在订单页面只是总金额发生了变化，我们还需要在订单页面把优惠券的信息也展示出来。

修改订单详情页：

_resources/views/orders/show.blade.php_

```
.
.
.
      <div class="order-summary text-right">
        <!-- 展示优惠信息开始 -->
        @if($order->couponCode)
        <div class="text-primary">
          <span>优惠信息：</span>
          <div class="value">{{ $order->couponCode->description }}</div>
        </div>
        @endif
        <!-- 展示优惠信息结束 -->
        <div class="total-amount">
          <span>订单总价：</span>
          <div class="value">￥{{ $order->total_amount }}</div>
        </div>
    .
    .
    .
```

然后刷新下订单页面看看：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0EIVQrkCfd.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0EIVQrkCfd.png!large)

## 订单金额不满足优惠券最低金额

我们在下单前的检查并没有判断订单金额是否满足优惠券最低金额，如果用户提交的商品总金额不满足优惠券的最低金额，则当提交订单时会抛出异常，这里我们需要把这个异常信息展示给用户。

修改购物车页面：

_resources/views/cart/index.blade.php_

```
.
.
.
@section('scriptsAfterJs')
<script>
  $(document).ready(function () {
    .
    .
    .
    $('.btn-create-order').click(function () {
      .
      .
      .
      axios.post('{{ route('orders.store') }}', req)
        .then(function (response) {
          .
          .
          .
        }, function (error) {
          if (error.response.status === 422) {
            .
            .
            .
          } else if (error.response.status === 403) { // 这里判断状态 403
            swal(error.response.data.msg, '', 'error');
          } else {
            swal('系统错误', '', 'error');
          }
        });
    });
    .
    .
    .
  })
</script>
@endsection
```

然后把 1 分钱的商品加到购物车，再在后台找一个固定金额的优惠券填入：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/yQzYy0s2NW.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/yQzYy0s2NW.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/x40fsMvNpr.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/x40fsMvNpr.png!large)

提交订单时报错，符合预期。

## 关闭订单减少优惠券用量

在自动关闭订单时，如果有使用优惠券则将该优惠券的用量减少：

_app/Jobs/CloseOrder.php_

```
.
.
.
    public function handle()
    {
        if ($this->order->paid_at) {
            return;
        }
        \DB::transaction(function () {
            .
            .
            .
            if ($this->order->couponCode) {
                $this->order->couponCode->changeUsed(false);
            }
        });
    }
```

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "优化优惠券模块"
```



