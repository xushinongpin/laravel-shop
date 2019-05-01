## 支付宝退款

上一节我们完成了拒绝退款的逻辑，这一节我们要实现同意退款的逻辑。

## 1. 生成退款订单号

不管是支付宝还是微信，在申请退款的时候都需要我们提交一个唯一字符串作为退款订单号，之后可以通过退款订单号来查询退款进度，退款的回调也会带上退款订单号。

我们选择在`Order`模型中写这个逻辑：

_app\Models\Order.php_

```
use Ramsey\Uuid\Uuid;
.
.
.
    public static function getAvailableRefundNo()
    {
        do {
            // Uuid类可以用来生成大概率不重复的字符串
            $no = Uuid::uuid4()->getHex();
            // 为了避免重复我们在生成之后在数据库中查询看看是否已经存在相同的退款订单号
        } while (self::query()->where('refund_no', $no)->exists());

        return $no;
    }
```

## 2. 控制器

接下来我们要完善一下我们之前在`OrdersController`里的`handleRefund()`方法，由于调用退款的逻辑比较多，因此我们单独拆出一个方法`_refundOrder()`来处理：

_app/Admin/Controllers/OrdersController.php_

```
use App\Exceptions\InternalException;
.
.
.
    public function handleRefund(Order $order, HandleRefundRequest $request)
    {
        if ($order->refund_status !== Order::REFUND_STATUS_APPLIED) {
            throw new InvalidRequestException('订单状态不正确');
        }
        if ($request->input('agree')) {
            // 清空拒绝退款理
            $extra = $order->extra ?: [];
            unset($extra['refund_disagree_reason']);
            $order->update([
                'extra' => $extra,
            ]);
            // 调用退款逻辑
            $this->_refundOrder($order);
        } else {
            .
            .
            .
        }

        return $order;
    }

    protected function _refundOrder(Order $order)
    {
        // 判断该订单的支付方式
        switch ($order->payment_method) {
            case 'wechat':
                // 微信的先留空
                // todo
                break;
            case 'alipay':
                // 用我们刚刚写的方法来生成一个退款订单号
                $refundNo = Order::getAvailableRefundNo();
                // 调用支付宝支付实例的 refund 方法
                $ret = app('alipay')->refund([
                    'out_trade_no' => $order->no, // 之前的订单流水号
                    'refund_amount' => $order->total_amount, // 退款金额，单位元
                    'out_request_no' => $refundNo, // 退款订单号
                ]);
                // 根据支付宝的文档，如果返回值里有 sub_code 字段说明退款失败
                if ($ret->sub_code) {
                    // 将退款失败的保存存入 extra 字段
                    $extra = $order->extra;
                    $extra['refund_failed_code'] = $ret->sub_code;
                    // 将订单的退款状态标记为退款失败
                    $order->update([
                        'refund_no' => $refundNo,
                        'refund_status' => Order::REFUND_STATUS_FAILED,
                        'extra' => $extra,
                    ]);
                } else {
                    // 将订单的退款状态标记为退款成功并保存退款订单号
                    $order->update([
                        'refund_no' => $refundNo,
                        'refund_status' => Order::REFUND_STATUS_SUCCESS,
                    ]);
                }
                break;
            default:
                // 原则上不可能出现，这个只是为了代码健壮性
                throw new InternalException('未知订单支付方式：'.$order->payment_method);
                break;
        }
    }
```

## 3. 前端模板

接下来我们要实现前端的`同意`按钮的逻辑：

_resources/views/admin/orders/show.blade.php_

```
.
.
.
<script>
  $(document).ready(function() {
.
.
.
    // 同意 按钮的点击事件
    $('#btn-refund-agree').click(function() {
      swal({
        title: '确认要将款项退还给用户？',
        type: 'warning',
        showCancelButton: true,
        confirmButtonText: "确认",
        cancelButtonText: "取消",
        showLoaderOnConfirm: true,
        preConfirm: function() {
          return $.ajax({
            url: '{{ route('admin.orders.handle_refund', [$order->id]) }}',
            type: 'POST',
            data: JSON.stringify({
              agree: true, // 代表同意退款
              _token: LA.token,
            }),
            contentType: 'application/json',
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

接下来我们就要来测试一下支付宝的退款逻辑是否正确。

先在用户界面创建一笔订单，并用支付宝支付，然后申请退款，这个过程这里就不再叙述。

> 别忘了从 requestbin 复制回调数据并提交，这样才能让我们系统里记录到对应的支付宝订单信息。如果忘了怎么操作可以返回到 7.3 节查看。

记录下支付的金额：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Iidd9iMzJe.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Iidd9iMzJe.png!large)

现在我们查看一下沙盒账户的余额，访问  
[https://openhome.alipay.com/platform/appDaily.htm?tab=account](https://openhome.alipay.com/platform/appDaily.htm?tab=account)，在`买家信息`里面可以看到当前的余额，也记录下来：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/XgbdIlYZ2u.png!large "XgbdIlYZ2u.png!large")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/XgbdIlYZ2u.png!large)

接着我们访问后台对应的订单详情页，点击退款的`同意`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0TbwIfeWVO.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0TbwIfeWVO.png!large)

点击`确认`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/06kLVwprUe.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/06kLVwprUe.png!large)

提示操作成功。点击`OK`按钮，页面自动刷新：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0z4AiCoHa2.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/0z4AiCoHa2.png!large)

可以看到订单退款状态变成了已退款。

接下来再访问支付宝的沙盒页面查看余额：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/VZm9hJv6oL.png!large "VZm9hJv6oL.png!large")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/VZm9hJv6oL.png!large)

计算`1021682.15 - 1021485.15 = 197`与我们的订单金额相同，退款成功。

再访问用户端的订单详情页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/75gJ0ZK2T0.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/75gJ0ZK2T0.png!large)

状态也变成了退款成功。

## 5. 细节优化

退款成功之后，我们在后台的订单详情页里面还能看到发货的表单，这个不符合逻辑，需要把它隐藏掉：

_resources/views/admin/orders/show.blade.php_

```
.
.
.
@if($order->ship_status === \App\Models\Order::SHIP_STATUS_PENDING)
  <!-- 加上这个判断条件 -->
  @if($order->refund_status !== \App\Models\Order::REFUND_STATUS_SUCCESS)
  .
  .
  .
  <!-- 在 上一个 if 的 else 前放上 endif -->
  @endif
@else
.
.
.
@endif
.
.
.
```

刷新后台订单详情页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/gx5DmD09D5.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/gx5DmD09D5.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "后台管理 - 同意退款 - 支付宝退款"
```

# 私人秘方

1. 报错 cURL error 60: SSL certificate problem: unable to get local issuer certificate \(see http: curl.haxx.se libcurl c libcurl errors.html\)

```
解决方案
下载证书：https://curl.haxx.se/ca/cacert.pem
修改php.ini
curl.cainfo = D:\BtSoft\WebSoft\php\7.1\cacert.pem
```

1. [支付宝业务错误文档](https://docs.open.alipay.com/api_1/alipay.trade.refund)



