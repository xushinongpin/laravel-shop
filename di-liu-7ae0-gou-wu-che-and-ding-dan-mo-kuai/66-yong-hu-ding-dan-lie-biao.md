## 订单列表

上一节我们完成了将购物车中的商品创建成订单，本章节将要实现用户端的订单列表页。

## 1. 控制器

在`OrdersController`新增`index()`方法：

_app/Http/Controllers/OrdersController.php_

```
use Illuminate\Http\Request;
    .
    .
    .
    public function index(Request $request)
    {
        $orders = Order::query()
            // 使用 with 方法预加载，避免N + 1问题
            ->with(['items.product', 'items.productSku']) 
            ->where('user_id', $request->user()->id)
            ->orderBy('created_at', 'desc')
            ->paginate();

        return view('orders.index', ['orders' => $orders]);
    }
```

## 2. 前端模板

接下来创建一个新的前端模板：

```
$ mkdir -p resources/views/orders && touch resources/views/orders/index.blade.php
```

_resources/views/orders/index.blade.php_

```
@extends('layouts.app')
@section('title', '订单列表')

@section('content')
<div class="row">
<div class="col-lg-10 offset-lg-1">
<div class="card">
  <div class="card-header">订单列表</div>
  <div class="card-body">
    <ul class="list-group">
      @foreach($orders as $order)
        <li class="list-group-item">
          <div class="card">
            <div class="card-header">
              订单号：{{ $order->no }}
              <span class="float-right">{{ $order->created_at->format('Y-m-d H:i:s') }}</span>
            </div>
            <div class="card-body">
              <table class="table">
                <thead>
                <tr>
                  <th>商品信息</th>
                  <th class="text-center">单价</th>
                  <th class="text-center">数量</th>
                  <th class="text-center">订单总价</th>
                  <th class="text-center">状态</th>
                  <th class="text-center">操作</th>
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
                    <td class="sku-price text-center">￥{{ $item->price }}</td>
                    <td class="sku-amount text-center">{{ $item->amount }}</td>
                    @if($index === 0)
                      <td rowspan="{{ count($order->items) }}" class="text-center total-amount">￥{{ $order->total_amount }}</td>
                      <td rowspan="{{ count($order->items) }}" class="text-center">
                        @if($order->paid_at)
                          @if($order->refund_status === \App\Models\Order::REFUND_STATUS_PENDING)
                            已支付
                          @else
                            {{ \App\Models\Order::$refundStatusMap[$order->refund_status] }}
                          @endif
                        @elseif($order->closed)
                          已关闭
                        @else
                          未支付<br>
                          请于 {{ $order->created_at->addSeconds(config('app.order_ttl'))->format('H:i') }} 前完成支付<br>
                          否则订单将自动关闭
                        @endif
                      </td>
                      <td rowspan="{{ count($order->items) }}" class="text-center"><a class="btn btn-primary btn-sm" href="">查看订单</a></td>
                    @endif
                  </tr>
                @endforeach
              </table>
            </div>
          </div>
        </li>
      @endforeach
    </ul>
    <div class="float-right">{{ $orders->render() }}</div>
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
.orders-index-page {
  .list-group-item {
    font-size: 12px;
    border: none;
    padding: 0;
    margin-bottom: 15px;
    .panel {
      margin: 0;
      .card-body {
        padding: 0;
      }
    }
    .table {
      margin: 0;
      td[rowspan] {
        border-left: 1px solid #ddd;
      }
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
    .total-amount {
      font-weight: bolder;
    }
  }
}
```

## 3. 路由

然后添加对应的路由：

_routes/web.php_

```
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::get('orders', 'OrdersController@index')->name('orders.index');
});
```

## 4. 查看效果

现在访问[http://shop.test/orders](http://shop.test/orders)查看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/7CmT6fcE69.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/7CmT6fcE69.png!large)

## 5. 添加入口

接下来我们要在顶部的菜单里加入`我的订单`这个入口：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
<div class="dropdown-menu" aria-labelledby="navbarDropdown">
  <a href="{{ route('user_addresses.index') }}" class="dropdown-item">收货地址</a>
  <a href="{{ route('orders.index') }}" class="dropdown-item">我的订单</a>
  .
  .
  .
</div>
```

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/eyy82GNEvV.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/eyy82GNEvV.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 订单列表页"
```



