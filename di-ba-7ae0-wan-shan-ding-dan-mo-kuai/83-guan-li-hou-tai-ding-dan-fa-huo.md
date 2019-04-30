## 订单发货

上一节的订单详情页我们只是完成了展示功能，接下来我们要在详情页上添加发货的功能。

## 1. 控制器

在`OrdersController`里新增一个`ship()`方法作为发货的接口：

_app/Admin/Controllers/OrdersController.php_

```
use Illuminate\Http\Request;
use App\Exceptions\InvalidRequestException;
.
.
.
    public function ship(Order $order, Request $request)
    {
        // 判断当前订单是否已支付
        if (!$order->paid_at) {
            throw new InvalidRequestException('该订单未付款');
        }
        // 判断当前订单发货状态是否为未发货
        if ($order->ship_status !== Order::SHIP_STATUS_PENDING) {
            throw new InvalidRequestException('该订单已发货');
        }
        // Laravel 5.5 之后 validate 方法可以返回校验过的值
        $data = $this->validate($request, [
            'express_company' => ['required'],
            'express_no'      => ['required'],
        ], [], [
            'express_company' => '物流公司',
            'express_no'      => '物流单号',
        ]);
        // 将订单发货状态改为已发货，并存入物流信息
        $order->update([
            'ship_status' => Order::SHIP_STATUS_DELIVERED,
            // 我们在 Order 模型的 $casts 属性里指明了 ship_data 是一个数组
            // 因此这里可以直接把数组传过去
            'ship_data'   => $data, 
        ]);

        // 返回上一页
        return redirect()->back();
    }
```

## 2. 路由

_app/Admin/routes.php_

```
$router->post('orders/{order}/ship', 'OrdersController@ship')->name('admin.orders.ship');
```

## 3. 前端模板

接下来我们在后台的订单详情页上加入发货的表单：

_resources/views/admin/orders/show.blade.php_

```
.
.
.
    <table class="table table-bordered">
    .
    .
    .
      <tr>
        <td>订单金额：</td>
        <td>￥{{ $order->total_amount }}</td>
        <!-- 这里也新增了一个发货状态 -->
        <td>发货状态：</td>
        <td>{{ \App\Models\Order::$shipStatusMap[$order->ship_status] }}</td>
      </tr>
      <!-- 订单发货开始 -->
      <!-- 如果订单未发货，展示发货表单 -->
      @if($order->ship_status === \App\Models\Order::SHIP_STATUS_PENDING)
      <tr>
        <td colspan="4">
          <form action="{{ route('admin.orders.ship', [$order->id]) }}" method="post" class="form-inline">
            <!-- 别忘了 csrf token 字段 -->
            {{ csrf_field() }}
            <div class="form-group {{ $errors->has('express_company') ? 'has-error' : '' }}">
              <label for="express_company" class="control-label">物流公司</label>
              <input type="text" id="express_company" name="express_company" value="" class="form-control" placeholder="输入物流公司">
              @if($errors->has('express_company'))
                @foreach($errors->get('express_company') as $msg)
                  <span class="help-block">{{ $msg }}</span>
                @endforeach
              @endif
            </div>
            <div class="form-group {{ $errors->has('express_no') ? 'has-error' : '' }}">
              <label for="express_no" class="control-label">物流单号</label>
              <input type="text" id="express_no" name="express_no" value="" class="form-control" placeholder="输入物流单号">
              @if($errors->has('express_no'))
                @foreach($errors->get('express_no') as $msg)
                  <span class="help-block">{{ $msg }}</span>
                @endforeach
              @endif
            </div>
            <button type="submit" class="btn btn-success" id="ship-btn">发货</button>
          </form>
        </td>
      </tr>
      @else
      <!-- 否则展示物流公司和物流单号 -->
      <tr>
        <td>物流公司：</td>
        <td>{{ $order->ship_data['express_company'] }}</td>
        <td>物流单号：</td>
        <td>{{ $order->ship_data['express_no'] }}</td>
      </tr>
      @endif
      <!-- 订单发货结束 -->
      </tbody>
    </table>
    .
    .
    .
```

## 4. 测试

接下来我们进入一个订单详情页，可以看到下面已经有了物流表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uqeaooPpX4.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uqeaooPpX4.png!large)

两个都留空直接提交表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lvBI8A79qC.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lvBI8A79qC.png!large)

填入信息之后提交，可以看到页面刷新，`发货状态`变更为`已发货`，并且展示物流信息：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/PoespHjRr1.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/PoespHjRr1.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 订单发货"
```



