## 订单详情页

接下来我们要实现订单详情页面。

## 1. 控制器

在`OrdersController`中新增`show()`方法：

_app/Http/Controllers/OrdersController.php_

```
.
.
.
    public function show(Order $order, Request $request)
    {
        return view('orders.show', ['order' => $order->load(['items.productSku', 'items.product'])]);
    }
.
.
.
```

这里的`load()`方法与上一章节介绍的`with()`预加载方法有些类似，称为`延迟预加载`，不同点在于`load()`是在已经查询出来的模型上调用，而`with()`则是在 ORM 查询构造器上调用。

## 2. 创建前端模板

创建一个新的前端模板页面：

```
$ touch resources/views/orders/show.blade.php
```

_resources/views/orders/show.blade.php_

```
@extends('layouts.app')
@section('title', '查看订单')

@section('content')
<div class="row">
<div class="col-lg-10 offset-lg-1">
<div class="card">
  <div class="card-header">
    <h4>订单详情</h4>
  </div>
  <div class="card-body">
    <table class="table">
      <thead>
      <tr>
        <th>商品信息</th>
        <th class="text-center">单价</th>
        <th class="text-center">数量</th>
        <th class="text-right item-amount">小计</th>
      </tr>
      </thead>
      @foreach($order->items as $index => $item)
        <tr>
          <td class="product-info">
            <div class="preview">
              <a target="_blank" href="{{ route('products.show', [$item->product_id]) }}">
                <img src="{{ $item->product->image_url }}">
              </a>
            </div>
            <div>
              <span class="product-title">
                 <a target="_blank" href="{{ route('products.show', [$item->product_id]) }}">{{ $item->product->title }}</a>
              </span>
              <span class="sku-title">{{ $item->productSku->title }}</span>
            </div>
          </td>
          <td class="sku-price text-center vertical-middle">￥{{ $item->price }}</td>
          <td class="sku-amount text-center vertical-middle">{{ $item->amount }}</td>
          <td class="item-amount text-right vertical-middle">￥{{ number_format($item->price * $item->amount, 2, '.', '') }}</td>
        </tr>
      @endforeach
      <tr><td colspan="4"></td></tr>
    </table>
    <div class="order-bottom">
      <div class="order-info">
        <div class="line"><div class="line-label">收货地址：</div><div class="line-value">{{ join(' ', $order->address) }}</div></div>
        <div class="line"><div class="line-label">订单备注：</div><div class="line-value">{{ $order->remark ?: '-' }}</div></div>
        <div class="line"><div class="line-label">订单编号：</div><div class="line-value">{{ $order->no }}</div></div>
      </div>
      <div class="order-summary text-right">
        <div class="total-amount">
          <span>订单总价：</span>
          <div class="value">￥{{ $order->total_amount }}</div>
        </div>
        <div>
          <span>订单状态：</span>
          <div class="value">
            @if($order->paid_at)
              @if($order->refund_status === \App\Models\Order::REFUND_STATUS_PENDING)
                已支付
              @else
                {{ \App\Models\Order::$refundStatusMap[$order->refund_status] }}
              @endif
            @elseif($order->closed)
              已关闭
            @else
              未支付
            @endif
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
</div>
</div>
@endsection
```

然后是样式文件：

_resources/sass/app.scss_

```
.
.
.
.orders-show-page {
  font-size: 12px;
  .vertical-middle {
    vertical-align: middle;
  }
  .product-info {
    display: flex;
    flex-direction: row;
    .preview {
      img {
        max-width: 80px;
        max-height: 80px;
      }
      border: 1px solid #eee;
      margin-right: 5px;
    }
    .product-title, .sku-title {
      display: block;
    }
    .product-title > a {
      color: #3c3c3c;
    }
    .sku-title {
      color: #9e9e9e;
    }
  }
  .table {
    margin-bottom: 0;
  }
  .item-amount {
    padding-right: 20px;
    width: 200px;
  }
  .order-bottom {
    display: flex;
    flex-direction: row;
  }
  .order-info {
    width: 50%;
    .line {
      display: flex;
      flex-direction: row;
      .line-label {
        width: 80px;
        text-align: right;
      }
      .line-value {
        flex-shrink: 100;
      }
    }
    border-right: 1px solid #ddd;
  }
  .order-summary {
    width: 50%;
    font-family: Verdana,Tahoma,Helvetica,Arial;
    .total-amount {
      font-weight: bolder;
      font-size: 14px;
    }
    .value {
      display: inline-block;
      width: 150px;
      padding-right: 20px;
    }
  }
}
```

## 3. 路由

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
    Route::get('orders/{order}', 'OrdersController@show')->name('orders.show');
});
```

## 4. 查看效果

现在访问[http://shop.test/orders/1](http://shop.test/orders/1)查看：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/OXfMA6Q1pQ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/OXfMA6Q1pQ.png!large)

## 5. 添加入口

现在需要在订单列表页面添加详情页的入口，将原本的查看订单按钮替换掉：

_resources/views/orders/index.blade.php_

```
.
.
.
<a class="btn btn-primary btn-sm" href="{{ route('orders.show', ['order' => $order->id]) }}">查看订单</a>
.
.
.
```

以及购物车页面创建订单成功之后也需跳转到订单页面：

_resources/views/cart/index.blade.php_

```
.
.
.
swal('订单提交成功', '', 'success')
  .then(() => {
    location.href = '/orders/' + response.data.id;
  });
.
.
.
```

## 6. 权限控制

为了安全起见我们只允许订单的创建者可以看到对应的订单信息，这个需求可以通过授权策略类（Policy）来实现。

通过`make:policy`命令创建一个授权策略类：

```
$ php artisan make:policy OrderPolicy
```

_app/Policies/OrderPolicy.php_

```
<?php

namespace App\Policies;

use App\Models\Order;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

class OrderPolicy
{
    use HandlesAuthorization;

    public function own(User $user, Order $order)
    {
        return $order->user_id == $user->id;
    }
}
```

我们之前在`AuthServiceProvider`中已经定义过了模型策略的自动加载逻辑，因此这里就不需要再次注册了。

最后在`OrdersController@show()`中校验权限：

_appHttp/Controllers/OrdersController.php_

```
.
.
.
    public function show(Order $order, Request $request)
    {
        $this->authorize('own', $order);
        return view('orders.show', ['order' => $order->load(['items.productSku', 'items.product'])]);
    }
```

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 订单详情页"
```



