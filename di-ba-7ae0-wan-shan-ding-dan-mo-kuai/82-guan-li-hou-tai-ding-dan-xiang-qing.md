[![](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/Ou8f1XMCOB.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/Ou8f1XMCOB.png?imageView2/2/w/1240/h/0)

## 订单详情

接下来我们要实现后台展示订单详情。

## 1. 控制器

由于订单信息比较多，Laravel-Admin 的表单形式不能很好地满足需求，因此这里我们采用自定义页面的方式来展示订单。

在`OrdersController`里新增`show()`方法：

_app/Admin/Controllers/OrdersController.php_

```
.
.
.
    public function show(Order $order, Content $content)
    {
        return $content
            ->header('查看订单')
            // body 方法可以接受 Laravel 的视图作为参数
            ->body(view('admin.orders.show', ['order' => $order]));
    }
```

这样的效果就是页面顶部和左侧都还是 Laravel-Admin 原本的菜单，而页面主要内容就变成了我们这个模板视图渲染的内容了。

## 2. 前端模板

接下来我们来实现前端模板：

```
$ mkdir -p resources/views/admin/orders/ && touch resources/views/admin/orders/show.blade.php
```

_resources/views/admin/orders/show.blade.php_

```
<div class="box box-info">
  <div class="box-header with-border">
    <h3 class="box-title">订单流水号：{{ $order->no }}</h3>
    <div class="box-tools">
      <div class="btn-group float-right" style="margin-right: 10px">
        <a href="{{ route('admin.orders.index') }}" class="btn btn-sm btn-default"><i class="fa fa-list"></i> 列表</a>
      </div>
    </div>
  </div>
  <div class="box-body">
    <table class="table table-bordered">
      <tbody>
      <tr>
        <td>买家：</td>
        <td>{{ $order->user->name }}</td>
        <td>支付时间：</td>
        <td>{{ $order->paid_at->format('Y-m-d H:i:s') }}</td>
      </tr>
      <tr>
        <td>支付方式：</td>
        <td>{{ $order->payment_method }}</td>
        <td>支付渠道单号：</td>
        <td>{{ $order->payment_no }}</td>
      </tr>
      <tr>
        <td>收货地址</td>
        <td colspan="3">{{ $order->address['address'] }} {{ $order->address['zip'] }} {{ $order->address['contact_name'] }} {{ $order->address['contact_phone'] }}</td>
      </tr>
      <tr>
        <td rowspan="{{ $order->items->count() + 1 }}">商品列表</td>
        <td>商品名称</td>
        <td>单价</td>
        <td>数量</td>
      </tr>
      @foreach($order->items as $item)
      <tr>
        <td>{{ $item->product->title }} {{ $item->productSku->title }}</td>
        <td>￥{{ $item->price }}</td>
        <td>{{ $item->amount }}</td>
      </tr>
      @endforeach
      <tr>
        <td>订单金额：</td>
        <td colspan="3">￥{{ $order->total_amount }}</td>
      </tr>
      </tbody>
    </table>
  </div>
</div>
```

## 3. 路由

接下来添加对应的路由：

_app/Admin/routes.php_

```
$router->get('orders/{order}', 'OrdersController@show')->name('admin.orders.show');
```

## 4. 测试

接下来我们直接访问[http://shop.test/admin/orders/{订单 ID](http://shop.test/admin/orders/%7B%E8%AE%A2%E5%8D%95ID)} ：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/TeMxmHZYVU.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/TeMxmHZYVU.png!large)

进入订单列表页查看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201808/11/5320/H6WpG4CjpB.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201808/11/5320/H6WpG4CjpB.png?imageView2/2/w/1240/h/0)

点击眼睛按钮可以进入订单详情页。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 订单详情"
```



