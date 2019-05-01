## 检查优惠券

上一节我们完成了优惠券的增删改查功能，这一节我们要实现用户在购物车界面上输入优惠券并检查是否有效。

## 1. 新增控制器

我们新增一个控制器来提供优惠券的查询功能：

```
$ php artisan make:controller CouponCodesController
```

_app/Http/Controllers/CouponCodesController.php_

```
<?php

namespace App\Http\Controllers;

use App\Models\CouponCode;
use Carbon\Carbon;

class CouponCodesController extends Controller
{
   public function show($code)
    {
        // 判断优惠券是否存在
        if (!$record = CouponCode::where('code', $code)->first()) {
            abort(404);
        }

        // 如果优惠券没有启用，则等同于优惠券不存在
        if (!$record->enabled) {
            abort(404);
        }

        if ($record->total - $record->used <= 0) {
            return response()->json(['msg' => '该优惠券已被兑完'], 403);
        }

        if ($record->not_before && $record->not_before->gt(Carbon::now())) {
            return response()->json(['msg' => '该优惠券现在还不能使用'], 403);
        }

        if ($record->not_after && $record->not_after->lt(Carbon::now())) {
            return response()->json(['msg' => '该优惠券已过期'], 403);
        }

        return $record;
    }
}
```

其中`abort()`方法可以直接中断我们程序的运行，接受的参数会变成 Http 状态码返回。在这里如果用户输入的优惠码不存在或者是没有启用我们就返回 404 给用户。

## 2. 添加路由

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
    Route::get('coupon_codes/{code}', 'CouponCodesController@show')->name('coupon_codes.show');
});
```

## 3. 前端模板

接下来我们要在购物车页面添加优惠券组件：

_resources/views/cart/index.blade.php_

```
.
.
.
<div class="form-group row">
  <label class="col-form-label col-sm-3 text-md-right">备注</label>
  <div class="col-sm-9 col-md-7">
    <textarea name="remark" class="form-control" rows="3"></textarea>
  </div>
</div>
<!-- 优惠码开始 -->
<div class="form-group row">
  <label class="col-form-label col-sm-3 text-md-right">优惠码</label>
  <div class="col-sm-4">
    <input type="text" class="form-control" name="coupon_code">
    <span class="form-text text-muted" id="coupon_desc"></span>
  </div>
  <div class="col-sm-3">
    <button type="button" class="btn btn-success" id="btn-check-coupon">检查</button>
    <button type="button" class="btn btn-danger" style="display: none;" id="btn-cancel-coupon">取消</button>
  </div>
</div>
<!-- 优惠码结束 -->
<div class="form-group">
  <div class="offset-sm-3 col-sm-3">
    <button type="button" class="btn btn-primary btn-create-order">提交订单</button>
  </div>
</div>
.
.
.
```

可以看到我们实际上添加了两个按钮，一个`检查`和一个`取消`，其中`取消`按钮是隐藏的。

我们要实现的效果是：用户输入一个优惠码，点击`检查`按钮，如果优惠码不正确则弹框提示，如果优惠码可以使用则输出优惠信息、将优惠码的输入框禁用、隐藏`检查`按钮、展示`取消`按钮。而点击`取消`按钮则反之，隐藏优惠信息、将优惠码的输入框启用、显示`检查`按钮、隐藏`取消`按钮。

我们先来看看 Html 代码的效果，随便找个商品加入购物车并进入购物车页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/50kz5Z70o1.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/50kz5Z70o1.png!large)

接下来是我们来实现按钮点击逻辑：

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
    // 检查按钮点击事件
    $('#btn-check-coupon').click(function () {
      // 获取用户输入的优惠码
      var code = $('input[name=coupon_code]').val();
      // 如果没有输入则弹框提示
      if(!code) {
        swal('请输入优惠码', '', 'warning');
        return;
      }
      // 调用检查接口
      axios.get('/coupon_codes/' + encodeURIComponent(code))
        .then(function (response) {  // then 方法的第一个参数是回调，请求成功时会被调用
          $('#coupon_desc').text(response.data.description); // 输出优惠信息
          $('input[name=coupon_code]').prop('readonly', true); // 禁用输入框
          $('#btn-cancel-coupon').show(); // 显示 取消 按钮
          $('#btn-check-coupon').hide(); // 隐藏 检查 按钮
        }, function (error) {
          // 如果返回码是 404，说明优惠券不存在
          if(error.response.status === 404) {
            swal('优惠码不存在', '', 'error');
          } else if (error.response.status === 403) {
          // 如果返回码是 403，说明有其他条件不满足
            swal(error.response.data.msg, '', 'error');
          } else {
          // 其他错误
            swal('系统内部错误', '', 'error');
          }
        })
    });

    // 隐藏 按钮点击事件
    $('#btn-cancel-coupon').click(function () {
      $('#coupon_desc').text(''); // 隐藏优惠信息
      $('input[name=coupon_code]').prop('readonly', false);  // 启用输入框
      $('#btn-cancel-coupon').hide(); // 隐藏 取消 按钮
      $('#btn-check-coupon').show(); // 显示 检查 按钮
    });

  });
</script>
@endsection
```

## 4. 测试

接下来我们测试一下这部分功能，优惠码输入框留空，直接点击`检查`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qPl1alMoju.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qPl1alMoju.png!large)

输入一个不存在的优惠码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/pwPPkFO6Ch.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/pwPPkFO6Ch.png!large)

点击`检查`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/1XWHVjzDs6.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/1XWHVjzDs6.png!large)

输入`WELCOME`（就是我们自己添加的那个优惠券，并且将开始时间设置成了几天之后）：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/c6CX5o1fug.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/c6CX5o1fug.png!large)

然后去后台优惠券列表找一个之前生成的测试优惠券，复制优惠码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/CpXTcuV5HT.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/CpXTcuV5HT.png!large)

填入到购物车的优惠码输入框中并点击`检查`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/sdKN1fv9H5.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/sdKN1fv9H5.png!large)

再点击`取消`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/bZTuG7hrrZ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/bZTuG7hrrZ.png!large)

符合预期。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 检查优惠券"
```



