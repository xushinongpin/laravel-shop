## 完善购物车页面

我们已经实现了购物车页面展示商品，在实现下单功能之前我们还需要在页面上添加收货地址和备注信息的输入框。

首先需要在购物车页面里显示用户已有的收货地址列表，因此需要在控制器中获取并注入到模板中：

_app/Http/Controllers/CartController.php_

```
.
.
.
    public function index(Request $request)
    {
        $cartItems = $request->user()->cartItems()->with(['productSku.product'])->get();
        $addresses = $request->user()->addresses()->orderBy('last_used_at', 'desc')->get();

        return view('cart.index', ['cartItems' => $cartItems, 'addresses' => $addresses]);
    }
.
.
.
```

通常来说用户重复使用最近用过的地址概率比较大，因此我们在取地址的时候根据`last_used_at`最后一次使用时间倒序排序，这样用户体验会好一些。

然后在购物车页面加入地址选择框和备注框，放到之前商品列表的`<table>`标签下面：

_resources/views/cart/index.blade.php_

```
.
.
.
</table>
<!-- 开始 -->
<div>
  <form class="form-horizontal" role="form" id="order-form">
    <div class="form-group row">
      <label class="col-form-label col-sm-3 text-md-right">选择收货地址</label>
      <div class="col-sm-9 col-md-7">
        <select class="form-control" name="address">
          @foreach($addresses as $address)
            <option value="{{ $address->id }}">{{ $address->full_address }} {{ $address->contact_name }} {{ $address->contact_phone }}</option>
          @endforeach
        </select>
      </div>
    </div>
    <div class="form-group row">
      <label class="col-form-label col-sm-3 text-md-right">备注</label>
      <div class="col-sm-9 col-md-7">
        <textarea name="remark" class="form-control" rows="3"></textarea>
      </div>
    </div>
    <div class="form-group">
      <div class="offset-sm-3 col-sm-3">
        <button type="button" class="btn btn-primary btn-create-order">提交订单</button>
      </div>
    </div>
  </form>
</div>
<!-- 结束 -->
.
.
.
```

进入购物车页面看下效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/C8pp5nowvb.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/C8pp5nowvb.png!large)

## 创建订单

接下来我们要实现创建订单的逻辑。

## 1. 控制器

首先创建`OrdersController`：

```
$ php artisan make:controller OrdersController
```

创建订单时我们需要对用户提交的数据做校验，因此还需创建一个`OrderRequest`

```
$ php artisan make:request OrderRequest
```

_app/Http/requests/OrderRequest.php_

```
<?php

namespace App\Http\Requests;

use Illuminate\Validation\Rule;
use App\Models\ProductSku;

class OrderRequest extends Request
{
    public function rules()
    {
        return [
            // 判断用户提交的地址 ID 是否存在于数据库并且属于当前用户
            // 后面这个条件非常重要，否则恶意用户可以用不同的地址 ID 不断提交订单来遍历出平台所有用户的收货地址
            'address_id'     => [
                'required',
                Rule::exists('user_addresses', 'id')->where('user_id', $this->user()->id),
            ],
            'items'  => ['required', 'array'],
            'items.*.sku_id' => [ // 检查 items 数组下每一个子数组的 sku_id 参数
                'required',
                function ($attribute, $value, $fail) {
                    if (!$sku = ProductSku::find($value)) {
                        return $fail('该商品不存在');
                    }
                    if (!$sku->product->on_sale) {
                        return $fail('该商品未上架');
                    }
                    if ($sku->stock === 0) {
                        return $fail('该商品已售完');
                    }
                    // 获取当前索引
                    preg_match('/items\.(\d+)\.sku_id/', $attribute, $m);
                    $index = $m[1];
                    // 根据索引找到用户所提交的购买数量
                    $amount = $this->input('items')[$index]['amount'];
                    if ($amount > 0 && $amount > $sku->stock) {
                        return $fail('该商品库存不足');
                    }
                },
            ],
            'items.*.amount' => ['required', 'integer', 'min:1'],
        ];
    }
}
```

在检查`sku_id`时我们依然判断了对应 SKU 是否存在、商品是否上架、库存是否充足，因为用户在把商品加入购物车，再到下单时商品的各个状态都可能发生变化。

在检查库存时，我们需要获取用户想要购买的该 SKU 数量，我们可以通过匿名函数的第一个参数`$attribute`来获取当前 SKU 所在的数组索引，比如第一个 SKU 的`$attribute`就是`items.0.sku_id`，所以我们采用正则的方式将这个`0`提取出来，`$this->input('items')[0]['amount']`就是用户想购买的数量。

接下来我们来写具体的创建订单逻辑：

_app/Http/Controllers/OrdersController.php_

```
<?php

namespace App\Http\Controllers;

use App\Http\Requests\OrderRequest;
use App\Models\ProductSku;
use App\Models\UserAddress;
use App\Models\Order;
use Carbon\Carbon;

class OrdersController extends Controller
{
    public function store(OrderRequest $request)
    {
        $user  = $request->user();
        // 开启一个数据库事务
        $order = \DB::transaction(function () use ($user, $request) {
            $address = UserAddress::find($request->input('address_id'));
            // 更新此地址的最后使用时间
            $address->update(['last_used_at' => Carbon::now()]);
            // 创建一个订单
            $order   = new Order([
                'address'      => [ // 将地址信息放入订单中
                    'address'       => $address->full_address,
                    'zip'           => $address->zip,
                    'contact_name'  => $address->contact_name,
                    'contact_phone' => $address->contact_phone,
                ],
                'remark'       => $request->input('remark'),
                'total_amount' => 0,
            ]);
            // 订单关联到当前用户
            $order->user()->associate($user);
            // 写入数据库
            $order->save();

            $totalAmount = 0;
            $items       = $request->input('items');
            // 遍历用户提交的 SKU
            foreach ($items as $data) {
                $sku  = ProductSku::find($data['sku_id']);
                // 创建一个 OrderItem 并直接与当前订单关联
                $item = $order->items()->make([
                    'amount' => $data['amount'],
                    'price'  => $sku->price,
                ]);
                $item->product()->associate($sku->product_id);
                $item->productSku()->associate($sku);
                $item->save();
                $totalAmount += $sku->price * $data['amount'];
            }

            // 更新订单总金额
            $order->update(['total_amount' => $totalAmount]);

            // 将下单的商品从购物车中移除
            $skuIds = collect($items)->pluck('sku_id');
            $user->cartItems()->whereIn('product_sku_id', $skuIds)->delete();

            return $order;
        });

        return $order;
    }
}
```

代码解析：

* `DB::transaction()`
  方法会开启一个数据库事务，在回调函数里的所有 SQL 写操作都会被包含在这个事务里，如果回调函数抛出异常则会自动回滚这个事务，否则提交事务。用这个方法可以帮我们节省不少代码。
* 在事务里先创建了一个订单，把当前用户设为订单的用户，然后把传入的地址数据快照进
  `address`
  字段。
* 然后遍历传入的商品 SKU 及其数量，
  `$order-`
  `>`
  `items()-`
  `>`
  `make()`
  方法可以新建一个关联关系的对象（也就是
  `OrderItem`
  ）但不保存到数据库，这个方法等同于
  `$item = new OrderItem(); $item-`
  `>`
  `order()-`
  `>`
  `associate($order);`
  。
* 然后根据所有的商品单价和数量求得订单的总价格，更新到刚刚创建的订单的
  `total_amount`
  字段。
* 最后使用 Laravel 提供的
  `collect()`
  辅助函数快速取得所有 SKU ID，然后将本次订单中的商品 SKU 从购物车中删除。

## 2. 减库存

写到这里其实还差一个很重要的步骤，那就是减库存。当用户下单时，我们需要立刻减少对应的商品库存，这样可以避免超卖的问题。

减库存不能简单地通过`update(['stock' => $sku->stock - $amount])`来操作，在高并发的情况下会有问题，这就需要通过数据库的方式来解决。

在`ProductSku`模型里新增两个方法：

_app/Models/ProductSku.php_

```
use App\Exceptions\InternalException;
    .
    .
    .
    public function decreaseStock($amount)
    {
        if ($amount < 0) {
            throw new InternalException('减库存不可小于0');
        }

        return $this->where('id', $this->id)->where('stock', '>=', $amount)->decrement('stock', $amount);
    }

    public function addStock($amount)
    {
        if ($amount < 0) {
            throw new InternalException('加库存不可小于0');
        }
        $this->increment('stock', $amount);
    }
```

代码解析：

* `decreaseStock()`
  方法里我们用了
  `decrement()`
  方法来减少字段的值，该方法会返回影响的行数。
* 最终执行的 SQL 类似于
  `update product_skus set stock = stock - $amount where id = $id and stock `
  `>`
  `= $amount`
  ，这样可以保证不会出现执行之后
  `stock`
  值为负数的情况，也就避免了超卖的问题。而且我们可以通过检查
  `decrement()`
  方法返回的影响行数来判断减库存操作是否成功，如果不成功说明商品库存不足。
* `addStock()`
  加库存的逻辑里面不需要像减库存那样判断了，但仍需通过
  `increment()`
  方法来保证操作的原子性。

接下来完善控制器中逻辑：

_app/Http/Controllers/OrdersController.php_

```
use App\Exceptions\InvalidRequestException;
.
.
.
foreach ($items as $data) {
    .
    .
    .
    $totalAmount += $sku->price * $data['amount'];
    if ($sku->decreaseStock($data['amount']) <= 0) {
        throw new InvalidRequestException('该商品库存不足');
    }
}
.
.
.
```

如果减库存失败则抛出异常，由于这块代码是在`DB::transaction()`中执行的，因此抛出异常时会触发事务的回滚，之前创建的`orders`和`order_items`记录都会被撤销。



## 3. 路由

接下来把控制器加入到路由中：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::post('orders', 'OrdersController@store')->name('orders.store');
}
```

## 4. 修改前端模板

接下来我们需要在购物车页面完善提交订单的逻辑：

_resources/views/cart/index.blade.php_

```
.
.
.
$(document).ready(function () {
  .
  .
  .
    // 监听创建订单按钮的点击事件
    $('.btn-create-order').click(function () {
      // 构建请求参数，将用户选择的地址的 id 和备注内容写入请求参数
      var req = {
        address_id: $('#order-form').find('select[name=address]').val(),
        items: [],
        remark: $('#order-form').find('textarea[name=remark]').val(),
      };
      // 遍历 <table> 标签内所有带有 data-id 属性的 <tr> 标签，也就是每一个购物车中的商品 SKU
      $('table tr[data-id]').each(function () {
        // 获取当前行的单选框
        var $checkbox = $(this).find('input[name=select][type=checkbox]');
        // 如果单选框被禁用或者没有被选中则跳过
        if ($checkbox.prop('disabled') || !$checkbox.prop('checked')) {
          return;
        }
        // 获取当前行中数量输入框
        var $input = $(this).find('input[name=amount]');
        // 如果用户将数量设为 0 或者不是一个数字，则也跳过
        if ($input.val() == 0 || isNaN($input.val())) {
          return;
        }
        // 把 SKU id 和数量存入请求参数数组中
        req.items.push({
          sku_id: $(this).data('id'),
          amount: $input.val(),
        })
      });
      axios.post('{{ route('orders.store') }}', req)
        .then(function (response) {
          swal('订单提交成功', '', 'success');
        }, function (error) {
          if (error.response.status === 422) {
            // http 状态码为 422 代表用户输入校验失败
            var html = '<div>';
            _.each(error.response.data.errors, function (errors) {
              _.each(errors, function (error) {
                html += error+'<br>';
              })
            });
            html += '</div>';
            swal({content: $(html)[0], icon: 'error'})
          } else {
            // 其他情况应该是系统挂了
            swal('系统错误', '', 'error');
          }
        });
    });

  });
```

## 5. 测试

接下来我们来测试一下提交订单的功能，为了尽可能地测试我们的代码，我们在购物车中放了 3 件商品，其中一件被下架，另外只勾选一件商品：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/dQMnLD0Ulz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/dQMnLD0Ulz.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/GW9IB0u7nX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/GW9IB0u7nX.png!large)

提交之后我们到数据库看看插入是否成功：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/8OtIOaC7T9.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/8OtIOaC7T9.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/q3ENs6t7Jf.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/q3ENs6t7Jf.png!large)

然后我们刷新购物车的界面，看看对应的商品是否已经从购物车中删除：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/kXdcFSXBuY.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/kXdcFSXBuY.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "创建订单"
```



