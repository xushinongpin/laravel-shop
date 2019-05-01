## 完善假数据填充

在之前的开发过程中我们只编写了商品的 Seeder，其他的测试数据都是我们在 tinker 中直接调用`factory()`函数来生成的，这不利于我们之后的测试，因此需要把其他模型的 Seeder 也补全。

## 1. 用户假数据

Laravel 默认就已经创建好了 User 模型的工厂文件，因此只需要编写 Seeder 文件即可：

```
$ php artisan make:seed UsersSeeder
```

_database/seeds/UsersSeeder.php_

```
<?php

use Illuminate\Database\Seeder;

class UsersSeeder extends Seeder
{
    public function run()
    {
        // 通过 factory 方法生成 100 个用户并保存到数据库中
        factory(\App\Models\User::class, 100)->create();
    }
}
```

## 2. 收货地址假数据

之前我们已经完成了收货地址的工厂文件，因此只需要编写 Seeder 文件即可：

```
$ php artisan make:seed UserAddressesSeeder
```

_database/seeds/UserAddressesSeeder.php_

```
<?php

use App\Models\User;
use App\Models\UserAddress;
use Illuminate\Database\Seeder;

class UserAddressesSeeder extends Seeder
{
    public function run()
    {
        User::all()->each(function (User $user) {
            factory(UserAddress::class, random_int(1, 3))->create(['user_id' => $user->id]);
        });
    }
}
```

代码解析：

* `User::all()`
  从数据获取所有的用户（我们之前通过 UsersSeeder 生成了 100 条），并返回一个集合
  `Collection`
  。
* `-`
  `>`
  `each()`
  是
  `Collection`
  的一个方法，与
  `foreach`
  类似，循环集合中的每一个元素，将其作为参数传递给匿名函数，在这里集合里的元素都是
  `User`
  类型。
* `factory(UserAddress::class, random_int(1, 3))`
  对每一个用户，产生一个 1 - 3 的随机数作为我们要为个用户生成地址的个数。
* `create(['user_id' =`
  `>`
  ` $user-`
  `>`
  `id])`
  将随机生成的数据写入数据库，同时指定这批数据的
  `user_id`
  字段统一为当前循环的用户 ID。

## 3. 优惠券假数据

之前我们已经完成了优惠券的工厂文件，因此只需要编写 Seeder 文件即可：

```
$ php artisan make:seed CouponCodesSeeder
```

_database/seeds/CouponCodesSeeder.php_

```
<?php

use Illuminate\Database\Seeder;

class CouponCodesSeeder extends Seeder
{
    public function run()
    {
        factory(\App\Models\CouponCode::class, 20)->create();
    }
}
```

## 4. 订单假数据

我们在编写订单相关功能的过程中并没有编写订单的工厂文件，这里需要补充一下。

我们先来实现`Order`模型的工厂文件：

_database/factories/OrderFactory.php_

```
<?php

use App\Models\CouponCode;
use App\Models\Order;
use App\Models\User;
use Faker\Generator as Faker;

$factory->define(Order::class, function (Faker $faker) {
    // 随机取一个用户
    $user = User::query()->inRandomOrder()->first();
    // 随机取一个该用户的地址
    $address = $user->addresses()->inRandomOrder()->first();
    // 10% 的概率把订单标记为退款
    $refund = random_int(0, 10) < 1;
    // 随机生成发货状态
    $ship = $faker->randomElement(array_keys(Order::$shipStatusMap));
    // 优惠券
    $coupon = null;
    // 30% 概率该订单使用了优惠券
    if (random_int(0, 10) < 3) {
        // 为了避免出现逻辑错误，我们只选择没有最低金额限制的优惠券
        $coupon = CouponCode::query()->where('min_amount', 0)->inRandomOrder()->first();
        // 增加优惠券的使用量
        $coupon->changeUsed();
    }

    return [
        'address'        => [
            'address'       => $address->full_address,
            'zip'           => $address->zip,
            'contact_name'  => $address->contact_name,
            'contact_phone' => $address->contact_phone,
        ],
        'total_amount'   => 0, 
        'remark'         => $faker->sentence,
        'paid_at'        => $faker->dateTimeBetween('-30 days'), // 30天前到现在任意时间点
        'payment_method' => $faker->randomElement(['wechat', 'alipay']),
        'payment_no'     => $faker->uuid,
        'refund_status'  => $refund ? Order::REFUND_STATUS_SUCCESS : Order::REFUND_STATUS_PENDING,
        'refund_no'      => $refund ? Order::getAvailableRefundNo() : null,
        'closed'         => false,
        'reviewed'       => random_int(0, 10) > 2,
        'ship_status'    => $ship,
        'ship_data'      => $ship === Order::SHIP_STATUS_PENDING ? null : [
            'express_company' => $faker->company,
            'express_no'      => $faker->uuid,
        ],
        'extra'          => $refund ? ['refund_reason' => $faker->sentence] : [],
        'user_id'        => $user->id,
        'coupon_code_id' => $coupon ? $coupon->id : null,
    ];
});
```

这里我们把`total_amount`设为 0，是因为这个是由`OrderItem`的数据计算出来的，我们在 Seeder 中生成了`OrderItem`之后再更新此字段。

接下来是`OrderItem`的工厂文件

_database/factories/OrderItemFactory.php_

```
<?php

use App\Models\OrderItem;
use App\Models\Product;
use Faker\Generator as Faker;

$factory->define(OrderItem::class, function (Faker $faker) {
    // 从数据库随机取一条商品
    $product = Product::query()->where('on_sale', true)->inRandomOrder()->first();
    // 从该商品的 SKU 中随机取一条
    $sku = $product->skus()->inRandomOrder()->first();

    return [
        'amount'         => random_int(1, 5), // 购买数量随机 1 - 5 份
        'price'          => $sku->price,
        'rating'         => null,
        'review'         => null,
        'reviewed_at'    => null,
        'product_id'     => $product->id,
        'product_sku_id' => $sku->id,
    ];
});
```

这里并没有生成`rating`和`review`，因为这些数据需要从`Order`那边获得，同样需要在 Seeder 中再更新。

接下来是 Seeder 文件：

```
$ php artisan make:seed OrdersSeeder
```

_database/seeds/OrdersSeeder.php_

```
<?php

use App\Models\Order;
use App\Models\OrderItem;
use App\Models\Product;
use Illuminate\Database\Seeder;

class OrdersSeeder extends Seeder
{
    public function run()
    {
        // 获取 Faker 实例
        $faker = app(Faker\Generator::class);
        // 创建 100 笔订单
        $orders = factory(Order::class, 100)->create();
        // 被购买的商品，用于后面更新商品销量和评分
        $products = collect([]);
        foreach ($orders as $order) {
            // 每笔订单随机 1 - 3 个商品
            $items = factory(OrderItem::class, random_int(1, 3))->create([
                'order_id'    => $order->id,
                'rating'      => $order->reviewed ? random_int(1, 5) : null,  // 随机评分 1 - 5
                'review'      => $order->reviewed ? $faker->sentence : null,
                'reviewed_at' => $order->reviewed ? $faker->dateTimeBetween($order->paid_at) : null, // 评价时间不能早于支付时间
            ]);

            // 计算总价
            $total = $items->sum(function (OrderItem $item) {
                return $item->price * $item->amount;
            });

            // 如果有优惠券，则计算优惠后价格
            if ($order->couponCode) {
                $total = $order->couponCode->getAdjustedPrice($total);
            }

            // 更新订单总价
            $order->update([
                'total_amount' => $total,
            ]);

            // 将这笔订单的商品合并到商品集合中
            $products = $products->merge($items->pluck('product'));
        }

        // 根据商品 ID 过滤掉重复的商品
        $products->unique('id')->each(function (Product $product) {
            // 查出该商品的销量、评分、评价数
            $result = OrderItem::query()
                ->where('product_id', $product->id)
                ->whereHas('order', function ($query) {
                    $query->whereNotNull('paid_at');
                })
                ->first([
                    \DB::raw('count(*) as review_count'),
                    \DB::raw('avg(rating) as rating'),
                    \DB::raw('sum(amount) as sold_count'),
                ]);

            $product->update([
                'rating'       => $result->rating ?: 5, // 如果某个商品没有评分，则默认为 5 分
                'review_count' => $result->review_count,
                'sold_count'   => $result->sold_count,
            ]);
        });
    }
}
```

由于通过`factory()`函数生成的订单不会产生事件去触发我们的监听器，所以我们在生成数据之后手动更新了商品的评分、评价数和销量。

## 注册到 DatabaseSeeder

接下来我们要把刚刚的这些 Seeder 文件注册到`DatabaseSeeder`中：

_database/seeds/DatabaseSeeder.php_

```
<?php

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        $this->call(UsersSeeder::class);
        $this->call(UserAddressesSeeder::class);
        $this->call(ProductsSeeder::class);
        $this->call(CouponCodesSeeder::class);
        $this->call(OrdersSeeder::class);
    }
}
```

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "完善假数据填充"
```



