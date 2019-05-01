## 使用优惠券下单

接下来我们要实现功能是使用优惠券来下单，这是优惠券模块中最核心的部分。

## 1. 修改购物车前端模板

首先我们需要在购物车页面把优惠码与订单的其他信息一起提交上来：

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
        var req = {
          address_id: $('#order-form').find('select[name=address]').val(),
          items: [],
          remark: $('#order-form').find('textarea[name=remark]').val(),
          coupon_code: $('input[name=coupon_code]').val(), // 从优惠码输入框中获取优惠码
        };
        .
        .
        .
      });
      .
      .
      .
    })
  </script>
@endsection
```

## 2. 校验优惠码

虽然我们在提交订单之前就已经检查过一次优惠码，但是提交时需要再次检查，因为有可能在用户检查优惠码和提交的时间空档中优惠券被其他人兑完了，或者是运营人员修改了优惠码规则。

检查逻辑和之前的差不多，为了让代码更优雅，我们把检查逻辑抽取出来。

首先要创建一个新的异常类，当优惠券不满足使用条件时抛出这个异常。可以使用`make:exception`来创建：

```
$ php artisan make:exception CouponCodeUnavailableException
```

_app/Exceptions/CouponCodeUnavailableException.php_

```
<?php

namespace App\Exceptions;

use Illuminate\Http\Request;
use Exception;

class CouponCodeUnavailableException extends Exception
{
    public function __construct($message, int $code = 403)
    {
        parent::__construct($message, $code);
    }

    // 当这个异常被触发时，会调用 render 方法来输出给用户
    public function render(Request $request)
    {
        // 如果用户通过 Api 请求，则返回 JSON 格式的错误信息
        if ($request->expectsJson()) {
            return response()->json(['msg' => $this->message], $this->code);
        }
        // 否则返回上一页并带上错误信息
        return redirect()->back()->withErrors(['coupon_code' => $this->message]);
    }
}
```

由于这个异常属于用户触发的业务异常，因此不需要记录在日志里，把它配到`ExceptionHandler`的`$dontReport`属性里：

_app/Exceptions/Handler.php_

```
 protected $dontReport = [
        InvalidRequestException::class,
        CouponCodeUnavailableException::class,
    ];
```

接下来我们把这个检查优惠券是否可用的逻辑放在`CouponCode`这个模型类里：

_app/Models/CouponCode.php_

```
use Carbon\Carbon;
use App\Exceptions\CouponCodeUnavailableException;
.
.
.
    public function checkAvailable($orderAmount = null)
    {
        if (!$this->enabled) {
            throw new CouponCodeUnavailableException('优惠券不存在');
        }

        if ($this->total - $this->used <= 0) {
            throw new CouponCodeUnavailableException('该优惠券已被兑完');
        }

        if ($this->not_before && $this->not_before->gt(Carbon::now())) {
            throw new CouponCodeUnavailableException('该优惠券现在还不能使用');
        }

        if ($this->not_after && $this->not_after->lt(Carbon::now())) {
            throw new CouponCodeUnavailableException('该优惠券已过期');
        }

        if (!is_null($orderAmount) && $orderAmount < $this->min_amount) {
            throw new CouponCodeUnavailableException('订单金额不满足该优惠券最低金额');
        }
    }
```

这个`checkAvailable()`方法接受一个参数`$orderAmount`订单金额。为了兼容用户下单前的校验，如果传入的`$orderAmount`是`null`则不去检查是否满足订单最低金额。

接下来我们把之前校验的代码修改成调用`checkAvailable()`方法：

_app/Http/Controllers/CouponCodesController.php_

```
<?php

namespace App\Http\Controllers;

use App\Exceptions\CouponCodeUnavailableException;
use App\Models\CouponCode;

class CouponCodesController extends Controller
{
    public function show($code)
    {
        if (!$record = CouponCode::where('code', $code)->first()) {
            throw new CouponCodeUnavailableException('优惠券不存在');
        }

        $record->checkAvailable();

        return $record;
    }
}
```

假如用户输入的优惠券不满足使用条件，`checkAvailable()`方法就会抛出异常并终止后面的流程。

## 3. 计算优惠后金额

接下来我们还需要实现一个**计算优惠后金额**的逻辑，依然放在`CouponCode`模型中：

_app/Models/CouponCode.php_

```
.
.
.
    public function getAdjustedPrice($orderAmount)
    {
        // 固定金额
        if ($this->type === self::TYPE_FIXED) {
            // 为了保证系统健壮性，我们需要订单金额最少为 0.01 元
            return max(0.01, $orderAmount - $this->value);
        }

        return number_format($orderAmount * (100 - $this->value) / 100, 2, '.', '');
    }
```

## 4. 新增、减少用量

优惠券的用量和商品 SKU 的库存类似，用户下单时我们新增对应优惠券的用量，如果订单超时关闭则减少用量。

同样是在`CouponCode`模型中实现：

_app/Models/CouponCode.php_

```
.
.
.
    public function changeUsed($increase = true)
    {
        // 传入 true 代表新增用量，否则是减少用量
        if ($increase) {
            // 与检查 SKU 库存类似，这里需要检查当前用量是否已经超过总量
            return $this->where('id', $this->id)->where('used', '<', $this->total)->increment('used');
        } else {
            return $this->decrement('used');
        }
    }
```

## 5. 修改下单逻辑

接下来我们需要修改之前的下单逻辑，使其支持优惠券。

首先我们需要修改之前封装的`OrderService`类：

_app/Services/OrderService.php_

```
use App\Models\CouponCode;
use App\Exceptions\CouponCodeUnavailableException;
.
.
.
    // 添加了一个 $coupon 的参数，可以为 null
    public function store(User $user, UserAddress $address, $remark, $items, CouponCode $coupon = null)
    {
        // 如果传入了优惠券，则先检查是否可用
        if ($coupon) {
            // 但此时我们还没有计算出订单总金额，因此先不校验
            $coupon->checkAvailable();
        }
        // 注意这里把 $coupon 也放到了 use 中
        $order = \DB::transaction(function () use ($user, $address, $remark, $items, $coupon) {
            .
            .
            .
            // 遍历用户提交的 SKU
            foreach ($items as $data) {
                .
                .
                .
            }
            if ($coupon) {
                // 总金额已经计算出来了，检查是否符合优惠券规则
                $coupon->checkAvailable($totalAmount);
                // 把订单金额修改为优惠后的金额
                $totalAmount = $coupon->getAdjustedPrice($totalAmount);
                // 将订单与优惠券关联
                $order->couponCode()->associate($coupon);
                // 增加优惠券的用量，需判断返回值
                if ($coupon->changeUsed() <= 0) {
                    throw new CouponCodeUnavailableException('该优惠券已被兑完');
                }
            }
           .
           .
           .
            return $order;
        });
        .
        .
        .
    }
```

然后在`OrdersController`修改对`OrderService`的调用：

_app/Http/Controllers/OrdersController.php_

```
use App\Exceptions\CouponCodeUnavailableException;
use App\Models\CouponCode;
.
.
.
    public function store(OrderRequest $request, OrderService $orderService)
    {
        $user    = $request->user();
        $address = UserAddress::find($request->input('address_id'));
        $coupon  = null;

        // 如果用户提交了优惠码
        if ($code = $request->input('coupon_code')) {
            $coupon = CouponCode::where('code', $code)->first();
            if (!$coupon) {
                throw new CouponCodeUnavailableException('优惠券不存在');
            }
        }
        // 参数中加入 $coupon 变量
        return $orderService->store($user, $address, $request->input('remark'), $request->input('items'), $coupon);
    }
.
.
.
```

## 6. 测试

接下来我们要测试一下优惠券的逻辑是否正确。

先添加一件商品到购物车，并输入一个可用的优惠券，并提交订单：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/aFjl55VgYn.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/aFjl55VgYn.png!large)

自动跳转到订单页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/sWkn8dfQoE.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/sWkn8dfQoE.png!large)

可以看到订单的`合计`金额与商品金额不一致，`2985 = 3980 * 0.75`与我们优惠券的折扣一致。

再到管理后台的优惠券列表页面看看：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/7k97JTnxCH.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/7k97JTnxCH.png!large)

用量也增加了，符合预期。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 使用优惠券下单"
```



