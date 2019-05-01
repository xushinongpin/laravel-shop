## 拒绝退款

接下来我们要实现管理后台处理用户退款申请的功能，我们先来实现拒绝用户的退款申请。

## 1. 控制器

我们先创建一个`HandleRefundRequest`来校验运营人员处理退款的请求：

```
$ php artisan make:request Admin/HandleRefundRequest
```

_app/Http/Requests/Admin/HandleRefundRequest.php_

```
<?php

namespace App\Http\Requests\Admin;

use App\Http\Requests\Request;

class HandleRefundRequest extends Request
{
    public function rules()
    {
        return [
            'agree'  => ['required', 'boolean'],
            'reason' => ['required_if:agree,false'], // 拒绝退款时需要输入拒绝理由
        ];
    }
}
```

然后在`OrdersController`中添加`handleRefund()`方法作为处理退款的接口：

_app/Admin/Controllers/OrdersController.php_

```
use App\Http\Requests\Admin\HandleRefundRequest;
.
.
.
    public function handleRefund(Order $order, HandleRefundRequest $request)
    {
        // 判断订单状态是否正确
        if ($order->refund_status !== Order::REFUND_STATUS_APPLIED) {
            throw new InvalidRequestException('订单状态不正确');
        }
        // 是否同意退款
        if ($request->input('agree')) {
            // 同意退款的逻辑这里先留空
            // todo
        } else {
            // 将拒绝退款理由放到订单的 extra 字段中
            $extra = $order->extra ?: [];
            $extra['refund_disagree_reason'] = $request->input('reason');
            // 将订单的退款状态改为未退款
            $order->update([
                'refund_status' => Order::REFUND_STATUS_PENDING,
                'extra'         => $extra,
            ]);
        }

        return $order;
    }
```

## 2. 路由

然后添加对应的路由：

_app/Admin/routes.php_

```
$router->post('orders/{order}/refund', 'OrdersController@handleRefund')->name('admin.orders.handle_refund');
```

## 3. 前端模板

接下来我们需要在后台的订单详情页里添加处理退款的按钮，把退款信息放在表格的最下方：

_resources/views/admin/orders/show.blade.php_

```
.
.
.
<tbody>
  .
  .
  .
  @if($order->refund_status !== \App\Models\Order::REFUND_STATUS_PENDING)
  <tr>
    <td>退款状态：</td>
    <td colspan="2">{{ \App\Models\Order::$refundStatusMap[$order->refund_status] }}，理由：{{ $order->extra['refund_reason'] }}</td>
    <td>
      <!-- 如果订单退款状态是已申请，则展示处理按钮 -->
      @if($order->refund_status === \App\Models\Order::REFUND_STATUS_APPLIED)
      <button class="btn btn-sm btn-success" id="btn-refund-agree">同意</button>
      <button class="btn btn-sm btn-danger" id="btn-refund-disagree">不同意</button>
      @endif
    </td>
  </tr>
  @endif
</tbody>
.
.
.
<script>
$(document).ready(function() {
  // 不同意 按钮的点击事件
  $('#btn-refund-disagree').click(function() {
    // Laravel-Admin 使用的 SweetAlert 版本与我们在前台使用的版本不一样，因此参数也不太一样
    swal({
      title: '输入拒绝退款理由',
      input: 'text',
      showCancelButton: true,
      confirmButtonText: "确认",
      cancelButtonText: "取消",
      showLoaderOnConfirm: true,
      preConfirm: function(inputValue) {
        if (!inputValue) {
          swal('理由不能为空', '', 'error')
          return false;
        }
        // Laravel-Admin 没有 axios，使用 jQuery 的 ajax 方法来请求
        return $.ajax({
          url: '{{ route('admin.orders.handle_refund', [$order->id]) }}',
          type: 'POST',
          data: JSON.stringify({   // 将请求变成 JSON 字符串
            agree: false,  // 拒绝申请
            reason: inputValue,
            // 带上 CSRF Token
            // Laravel-Admin 页面里可以通过 LA.token 获得 CSRF Token
            _token: LA.token,
          }),
          contentType: 'application/json',  // 请求的数据格式为 JSON
        });
      },
      allowOutsideClick: false
    }).then(function (ret) {
      // 如果用户点击了『取消』按钮，则不做任何操作
      if (ret.dismiss === 'cancel') {
        return;
      }
      swal({
        title: '操作成功',
        type: 'success'
      }).then(function() {
        // 用户点击 swal 上的按钮时刷新页面
        location.reload();
      });
    });
  });
});
</script>
```

## 4. 测试

接下来我们测试一下，在后台的订单列表页面找到申请退款的那笔订单，进入详情页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uwAWWfo5mN.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uwAWWfo5mN.png!large)

点击`不同意`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/ZtzCFbkDV3.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/ZtzCFbkDV3.png!large)

输入拒绝退款理由，点击`确认`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/yY2jejgqXK.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/yY2jejgqXK.png!large)

点击`OK`之后页面自动刷新：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/IAd4z7ptVj.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/IAd4z7ptVj.png!large)

已经没有退款的信息了。

## 5. 细节优化

接下来我们需要在用户界面展示拒绝退款理由。

_resources/views/orders/show.blade.php_

```
.
.
.
        <div>
          <span>订单状态：</span>
          <div class="value">
            .
            .
            .
          </div>
        </div>
        @if(isset($order->extra['refund_disagree_reason']))
        <div>
          <span>拒绝退款理由：</span>
          <div class="value">{{ $order->extra['refund_disagree_reason'] }}</div>
        </div>
        @endif
.
.
.
```

再访问用户界面的订单详情页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/8SFLm9n3AX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/8SFLm9n3AX.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 拒绝退款"
```



