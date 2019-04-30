## 订单模块

订单是电商系统的核心之一，本章节将要实现把购物车中的商品提交成订单。

## 1. 整理字段

由于我们的一笔订单支持多个商品 SKU，因此我们需要`orders`和`order_items`两张表，`orders`保存用户、金额、收货地址等信息，`order_items`则保存商品 SKU ID、数量以及与`orders`表的关联。

我们先整理`orders`表的字段：

| 字段名称 | 描述 | 类型 | 加索引缘由 |
| :--- | :--- | :--- | :--- |
| id | 自增长 ID | unsigned big int | 主键 |
| no | 订单流水号 | varchar | 唯一 |
| user\_id | 下单的用户 ID | unsigned big int | 外键 |
| address | JSON 格式的收货地址 | text | 无 |
| total\_amount | 订单总金额 | decimal | 无 |
| remark | 订单备注 | text | 无 |
| paid\_at | 支付时间 | datetime, null | 无 |
| payment\_method | 支付方式 | varchar, null | 无 |
| payment\_no | 支付平台订单号 | varchar, null | 无 |
| refund\_status | 退款状态 | varchar | 无 |
| refund\_no | 退款单号 | varchar, null | 唯一 |
| closed | 订单是否已关闭 | tinyint, default 0 | 无 |
| reviewed | 订单是否已评价 | tinyint, default 0 | 无 |
| ship\_status | 物流状态 | varchar | 无 |
| ship\_data | 物流数据 | text, null | 无 |
| extra | 其他额外的数据 | text, null | 无 |

这里我们把收货地址用 JSON 格式保存而不是直接用一个外键连接到地址表，假如用户用地址 A 创建了一个订单，然后又修改了地址 A，那么用外键连接的方式这个订单的地址就会变成新地址，这并不符合正常的逻辑，所以我们需要用 JSON 格式把下单时的地址快照进订单，这样无论用户是修改还是删除之前的地址，都不会影响到之前的订单。

然后是`order_items`：

| 字段名称 | 描述 | 类型 | 加索引缘由 |
| :--- | :--- | :--- | :--- |
| id | 自增长 ID | unsigned big int | 主键 |
| order\_id | 所属订单 ID | unsigned big int | 外键 |
| product\_id | 对应商品 ID | unsigned big int | 外键 |
| product\_sku\_id | 对应商品 SKU ID | unsigned big int | 外键 |
| amount | 数量 | unsigned int | 无 |
| price | 单价 | decimal | 无 |
| rating | 用户打分 | unsigned int | 无 |
| review | 用户评价 | text | 无 |
| reviewed\_at | 评价时间 | timestamp, null | 无 |

## 2. 创建模型

```
$ php artisan make:model Models/Order -mf
$ php artisan make:model Models/OrderItem -mf
```

根据上面整理好的字段修改数据库迁移文件：

_database/migrations/&lt; your\_date &gt;\_create\_orders\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('no')->unique();
            $table->unsignedBigInteger('user_id');
            $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
            $table->text('address');
            $table->decimal('total_amount', 10, 2);
            $table->text('remark')->nullable();
            $table->dateTime('paid_at')->nullable();
            $table->string('payment_method')->nullable();
            $table->string('payment_no')->nullable();
            $table->string('refund_status');
            $table->string('refund_no')->unique()->nullable();
            $table->boolean('closed')->default(false);
            $table->boolean('reviewed')->default(false);
            $table->string('ship_status');
            $table->text('ship_data')->nullable();
            $table->text('extra')->nullable();
            $table->timestamps();
        });
    }
.
.
.
```

_database/migrations/&lt; your\_date &gt;\_create\_order\_items\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('order_items', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('order_id');
            $table->foreign('order_id')->references('id')->on('orders')->onDelete('cascade');
            $table->unsignedBigInteger('product_id');
            $table->foreign('product_id')->references('id')->on('products')->onDelete('cascade');
            $table->unsignedBigInteger('product_sku_id');
            $table->foreign('product_sku_id')->references('id')->on('product_skus')->onDelete('cascade');
            $table->unsignedInteger('amount');
            $table->decimal('price', 10, 2);
            $table->unsignedInteger('rating')->nullable();
            $table->text('review')->nullable();
            $table->timestamp('reviewed_at')->nullable();
        });
    }
.
.
.
```

然后修改模型文件，设置好各个属性以及关联关系：

_app/Models/Order.php_

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Order extends Model
{
    const REFUND_STATUS_PENDING = 'pending';
    const REFUND_STATUS_APPLIED = 'applied';
    const REFUND_STATUS_PROCESSING = 'processing';
    const REFUND_STATUS_SUCCESS = 'success';
    const REFUND_STATUS_FAILED = 'failed';

    const SHIP_STATUS_PENDING = 'pending';
    const SHIP_STATUS_DELIVERED = 'delivered';
    const SHIP_STATUS_RECEIVED = 'received';

    public static $refundStatusMap = [
        self::REFUND_STATUS_PENDING    => '未退款',
        self::REFUND_STATUS_APPLIED    => '已申请退款',
        self::REFUND_STATUS_PROCESSING => '退款中',
        self::REFUND_STATUS_SUCCESS    => '退款成功',
        self::REFUND_STATUS_FAILED     => '退款失败',
    ];

    public static $shipStatusMap = [
        self::SHIP_STATUS_PENDING   => '未发货',
        self::SHIP_STATUS_DELIVERED => '已发货',
        self::SHIP_STATUS_RECEIVED  => '已收货',
    ];

    protected $fillable = [
        'no',
        'address',
        'total_amount',
        'remark',
        'paid_at',
        'payment_method',
        'payment_no',
        'refund_status',
        'refund_no',
        'closed',
        'reviewed',
        'ship_status',
        'ship_data',
        'extra',
    ];

    protected $casts = [
        'closed'    => 'boolean',
        'reviewed'  => 'boolean',
        'address'   => 'json',
        'ship_data' => 'json',
        'extra'     => 'json',
    ];

    protected $dates = [
        'paid_at',
    ];

    protected static function boot()
    {
        parent::boot();
        // 监听模型创建事件，在写入数据库之前触发
        static::creating(function ($model) {
            // 如果模型的 no 字段为空
            if (!$model->no) {
                // 调用 findAvailableNo 生成订单流水号
                $model->no = static::findAvailableNo();
                // 如果生成失败，则终止创建订单
                if (!$model->no) {
                    return false;
                }
            }
        });
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function items()
    {
        return $this->hasMany(OrderItem::class);
    }

    public static function findAvailableNo()
    {
        // 订单流水号前缀
        $prefix = date('YmdHis');
        for ($i = 0; $i < 10; $i++) {
            // 随机生成 6 位的数字
            $no = $prefix.str_pad(random_int(0, 999999), 6, '0', STR_PAD_LEFT);
            // 判断是否已经存在
            if (!static::query()->where('no', $no)->exists()) {
                return $no;
            }
        }
        \Log::warning('find order no failed');

        return false;
    }
}
```

这里我们定义了`REFUND_*`和`SHIP_STATUS_*`两批常量，分别代表退款的各个状态和物流的各个状态，在我们之后的代码里涉及到修改这两个字段值的地方都要用这些常量来赋值。同时还定义了`$refundStatusMap`和`$shipStatusMap`两个静态数组，将上面定义好的常量和中文描述对应起来。

我们在`boot()`方法中注册了一个模型创建事件监听函数，用于自动生成订单的流水号。

定义好常量之后我们需要再调整一下刚刚的迁移文件，给退款状态和物流状态加上默认值，退款状态默认未退款，物流状态默认未发货：

_database/migrations/&lt; your\_date &gt;\_create\_orders\_table.php_

```
.
.
.
$table->string('refund_status')->default(\App\Models\Order::REFUND_STATUS_PENDING);
.
.
.
$table->string('ship_status')->default(\App\Models\Order::SHIP_STATUS_PENDING);
.
.
.
```

接下来是`OrderItem`模型：

_app/Models/OrderItem.php_

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class OrderItem extends Model
{
    protected $fillable = ['amount', 'price', 'rating', 'review', 'reviewed_at'];
    protected $dates = ['reviewed_at'];
    public $timestamps = false;

    public function product()
    {
        return $this->belongsTo(Product::class);
    }

    public function productSku()
    {
        return $this->belongsTo(ProductSku::class);
    }

    public function order()
    {
        return $this->belongsTo(Order::class);
    }
}
```

`public $timestamps = false;`代表这个模型没有`created_at`和`updated_at`两个时间戳字段。

最后执行迁移：

```
$ php artisan migrate
```

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "订单模型"
```



