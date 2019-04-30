## 购物车

购物车是电商网站一个必备的功能，本章节要实现的功能是将商品添加到购物。

购物车的数据通常会保存到 Session 或者数据库。对于拥有多个端（网页、App）的电商网站来说为了保障用户体验会使用数据库来保存购物车中的数据，这样用户在网页端加入购物车的商品也能在 App 中看到。本项目虽然目前只有一个网页端，但这是一个实战项目，是有可能拓展出 APP、小程序等其他端，因此我们选择数据库来保存购物车的数据。

## 整理字段

我们把购物车中的数据存入`cart_items`表，表结构如下：

| 字段名称 | 描述 | 类型 | 加索引缘由 |
| :--- | :--- | :--- | :--- |
| id | 自增长 ID | unsigned big int | 主键 |
| user\_id | 所属用户 ID | unsigned big int | 外键 |
| product\_sku\_id | 商品 SKU ID | unsigned big int | 外键 |
| amount | 商品数量 | unsigned int | 无 |

## 1. 模型

接下来我们创建对应的模型：

```
$ php artisan make:model Models/CartItem -m
```

根据上面整理的字段编辑数据库迁移文件：

_databases/migrations/&lt; your\_date &gt;\_create\_cart\_items\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('cart_items', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('user_id');
            $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
            $table->unsignedBigInteger('product_sku_id');
            $table->foreign('product_sku_id')->references('id')->on('product_skus')->onDelete('cascade');
            $table->unsignedInteger('amount');
        });
    }
.
.
.
```

_app/Models/CartItem.php_

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class CartItem extends Model
{
    protected $fillable = ['amount'];
    public $timestamps = false;

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function productSku()
    {
        return $this->belongsTo(ProductSku::class);
    }
}
```

然后在`User`模型中加上关联关系：

_app/Models/User.php_

```
.
.
.
    public function cartItems()
    {
        return $this->hasMany(CartItem::class);
    }
.
.
.
```

最后执行数据库迁移：

```
$ php artisan migrate
```

## 1. 控制器

首先创建`CartController`：

```
$ php artisan make:controller CartController
```

然后创建一个`AddCartRequest`：

```
$ php artisan make:request AddCartRequest
```

我们将商品添加到购物车的时候会提交两个参数：

1. 商品 SKU ID；
2. 数量。

因此需要在`AddCartRequest`中校验这两个参数：

_app/Http/Requests/AddCartRequest.php_

```
<?php

namespace App\Http\Requests;

use App\Models\ProductSku;

class AddCartRequest extends Request
{
    public function rules()
    {
        return [
            'sku_id' => [
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
                    if ($this->input('amount') > 0 && $sku->stock < $this->input('amount')) {
                        return $fail('该商品库存不足');
                    }
                },
            ],
            'amount' => ['required', 'integer', 'min:1'],
        ];
    }

    public function attributes()
    {
        return [
            'amount' => '商品数量'
        ];
    }

    public function messages()
    {
        return [
            'sku_id.required' => '请选择商品'
        ];
    }
}
```

校验`sku_id`的第二个规则是一个闭包校验规则，这个闭包接受 3 个参数，分别是参数名、参数值和错误回调。在这个闭包里我们依次判断了用户提交的 SKU ID 是否存在、商品是否上架以及库存是否充足。

> 闭包校验规则允许我们直接通过匿名函数的方式来校验用户输入，比较适合在项目中只使用一次的情况。闭包校验规则在 Laravel 5.5 开始支持，但在 5.6 的文档才有介绍。

接下来我们在`CartController`添加一个`add()`方法，参数就是我们刚刚创建的`AddCartRequest`

_app/Http/Controllers/CartController.php_

```
use App\Http\Requests\AddCartRequest;
use App\Models\CartItem;
.
.
.
    public function add(AddCartRequest $request)
    {
        $user   = $request->user();
        $skuId  = $request->input('sku_id');
        $amount = $request->input('amount');

        // 从数据库中查询该商品是否已经在购物车中
        if ($cart = $user->cartItems()->where('product_sku_id', $skuId)->first()) {

            // 如果存在则直接叠加商品数量
            $cart->update([
                'amount' => $cart->amount + $amount,
            ]);
        } else {

            // 否则创建一个新的购物车记录
            $cart = new CartItem(['amount' => $amount]);
            $cart->user()->associate($user);
            $cart->productSku()->associate($skuId);
            $cart->save();
        }

        return [];
    }
```

## 2. 路由

接下来把这个控制器加入到路由中：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function () {
    .
    .
    .
    Route::post('cart', 'CartController@add')->name('cart.add');
});
```

## 3. 前端模板

接下来我们要实现前端模板页面`加入购物车`按钮的逻辑：

_resources/views/products/show.blade.php_

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
    // 加入购物车按钮点击事件
    $('.btn-add-to-cart').click(function () {

      // 请求加入购物车接口
      axios.post('{{ route('cart.add') }}', {
        sku_id: $('label.active input[name=skus]').val(),
        amount: $('.cart_amount input').val(),
      })
        .then(function () { // 请求成功执行此回调
          swal('加入购物车成功', '', 'success');
        }, function (error) { // 请求失败执行此回调
          if (error.response.status === 401) {

            // http 状态码为 401 代表用户未登陆
            swal('请先登录', '', 'error');

          } else if (error.response.status === 422) {

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
        })
    });

  });
```

代码解析：

* 当用户点击
  `加入购物车`
  按钮时，通过
  `$('label.active input[name=skus]')`
  这个 CSS 选择器取得当前被选中的 SKU，并取得对应的 ID。
* 如果后端返回的 Http 状态是
  `200`
  ，则进入到
  `then()`
  方法的第一个回调函数里，即弹框告知用户操作成功。
* 如果后端返回的 Http 状态不是
  `200`
  ，则进入到
  `then()`
  方法的第二个回调函数里，可以通过
  `error.response.status`
  来取得 Http 状态码，返回的数据可以通过
  `error.response.data`
  来取得。
* 在 Laravel 里输入参数校验不通过抛出的异常所对应的 Http 状态码是
  `422`
  ，具体错误信息会放在返回结果的
  `errors`
  数组里，所以这里我们通过
  `error.response.data.errors`
  来拿到所有的错误信息。最后把所有的错误信息拼接成 Html 代码并弹框告知用户。
* 如果状态码既不是
  `200`
  也不是
  `422`
  ，那说明是我们系统其他地方出问题了，直接弹框告知用户系统错误。

## 4. 测试

现在我们来看一下效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/IrPC0Bwtfj.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/IrPC0Bwtfj.gif!large)

然后到数据库中查看是否有购物车记录：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/73UtNx80Uq.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/73UtNx80Uq.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "商品加入购物车"
```

403解决方案

```
非教程方案，自己找的解决方法
Http/Requests/AddCartRequest.php 这个文件添加权限
public function authorize()
{
return true;
}

```



