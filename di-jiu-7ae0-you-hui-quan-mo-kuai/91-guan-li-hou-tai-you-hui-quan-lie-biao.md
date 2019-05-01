[![](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/XHjhRU2GWh.jpeg?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/XHjhRU2GWh.jpeg?imageView2/2/w/1240/h/0)

## 优惠券

优惠券模块是电商系统里一个重要的模块。本章节将要实现优惠券的数据库表创建并在管理后台查看优惠券列表。

## 1. 整理字段

在创建数据库表之前，我们需要先分析一下优惠券的表应该有哪些字段：

| 字段名称 | 描述 | 类型 | 加索引缘由 |
| :--- | :--- | :--- | :--- |
| id | 自增长 ID | unsigned big int | 主键 |
| name | 优惠券的标题 | varchar | 无 |
| code | 优惠码，用户下单时输入 | varchar | 唯一 |
| type | 优惠券类型，支持固定金额和百分比折扣 | varchar | 无 |
| value | 折扣值，根据不同类型含义不同 | decimal | 无 |
| total | 全站可兑换的数量 | unsigned int | 无 |
| used | 当前已兑换的数量 | unsigned int, default 0 | 无 |
| min\_amount | 使用该优惠券的最低订单金额 | decimal | 无 |
| not\_before | 在这个时间之前不可用 | datetime, null | 无 |
| not\_after | 在这个时间之后不可用 | datetime, null | 无 |
| enabled | 优惠券是否生效 | tinyint | 无 |

## 2. 创建模型

接下来我们创建一个`CouponCode`模型：

```
$ php artisan make:model Models/CouponCode -mf
```

根据上面整理好的字段修改数据库迁移文件：

_database/migrations/&lt; your\_date &gt;\_create\_coupon\_codes\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('coupon_codes', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('name');
            $table->string('code')->unique();
            $table->string('type');
            $table->decimal('value');
            $table->unsignedInteger('total');
            $table->unsignedInteger('used')->default(0);
            $table->decimal('min_amount', 10, 2);
            $table->datetime('not_before')->nullable();
            $table->datetime('not_after')->nullable();
            $table->boolean('enabled');
            $table->timestamps();
        });
    }
```

然后修改模型文件，加入一些必要的字段信息：

_app/Models/CouponCode.php_

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class CouponCode extends Model
{
    // 用常量的方式定义支持的优惠券类型
    const TYPE_FIXED = 'fixed';
    const TYPE_PERCENT = 'percent';

    public static $typeMap = [
        self::TYPE_FIXED   => '固定金额',
        self::TYPE_PERCENT => '比例',
    ];

    protected $fillable = [
        'name',
        'code',
        'type',
        'value',
        'total',
        'used',
        'min_amount',
        'not_before',
        'not_after',
        'enabled',
    ];
    protected $casts = [
        'enabled' => 'boolean',
    ];
    // 指明这两个字段是日期类型
    protected $dates = ['not_before', 'not_after'];
}
```

同时我们还需要修改`orders`表，在里面添加一个`coupon_code_id`字段。

创建一个数据库迁移文件：

```
$ php artisan make:migration orders_add_coupon_code_id --table=orders
```

_database/migrations/&lt; your\_date &gt;\_orders\_add\_coupon\_code\_id.php_

```
.
.
.
    public function up()
    {
        Schema::table('orders', function (Blueprint $table) {
            $table->unsignedBigInteger('coupon_code_id')->nullable()->after('paid_at');
            $table->foreign('coupon_code_id')->references('id')->on('coupon_codes')->onDelete('set null');
        });
    }

    public function down()
    {
        Schema::table('orders', function (Blueprint $table) {
            $table->dropForeign(['coupon_code_id']);
            $table->dropColumn('coupon_code_id');
        });
    }
```

代码解析：

* `onDelete('set null')`
  代表如果这个订单有关联优惠券并且该优惠券被删除时将自动把
  `coupon_code_id`
  设成
  `null`
  。我们不能因为删除了优惠券就把关联了这个优惠券的订单都删除了，这是绝对不允许的。
* `dropForeign()`
  删除外键关联，要早于
  `dropColumn()`
  删除字段调用，否则数据库会报错。
* `dropForeign()`
  方法的参数可以是字符串也可以是一个数组，如果是字符串则代表删除外键名为该字符串的外键，而如果是数组的话则会删除该数组中字段所对应的外键。我们这个
  `coupon_code_id`
  字段默认的外键名是
  `orders_coupon_code_id_foreign`
  ，因此需要通过数组的方式来删除。

接下来是在`Order`模型中新增与`CouponCode`的关联关系：

_app/Models/Order.php_

```
.
.
.
    public function couponCode()
    {
        return $this->belongsTo(CouponCode::class);
    }
.
.
.
```

最后执行数据库迁移：

```
$ php artisan migrate
```

## 3. 创建控制器

接下来我们要创建一个管理后台的控制器，使用`admin:make`来自动生成：

```
$ php artisan admin:make CouponCodesController --model=App\\Models\\CouponCode
```

本章节要实现的是优惠券列表，因此只需要关注`index()`和`grid()`两个方法：

_app/Admin/Controllers/CouponCodesController.php_

```
.
.
.
    public function index(Content $content)
    {
        return $content
            ->header('优惠券列表')
            ->body($this->grid());
    }
    .
    .
    .
    protected function grid()
    {
        $grid = new Grid(new CouponCode);

        // 默认按创建时间倒序排序
        $grid->model()->orderBy('created_at', 'desc');
        $grid->id('ID')->sortable();
        $grid->name('名称');
        $grid->code('优惠码');
        $grid->type('类型')->display(function($value) {
            return CouponCode::$typeMap[$value];
        });
        // 根据不同的折扣类型用对应的方式来展示
        $grid->value('折扣')->display(function($value) {
            return $this->type === CouponCode::TYPE_FIXED ? '￥'.$value : $value.'%';
        });
        $grid->min_amount('最低金额');
        $grid->total('总量');
        $grid->used('已用');
        $grid->enabled('是否启用')->display(function($value) {
            return $value ? '是' : '否';
        });
        $grid->created_at('创建时间');

        $grid->actions(function ($actions) {
            $actions->disableView();
        });

        return $grid;
    }
.
.
.
```

在后台我们不需要优惠券详情页，因此删除`show()`和`detail()`两个方法。

## 4. 添加路由

然后添加对应路由：

_app/Admin/routes.php_

```
$router->get('coupon_codes', 'CouponCodesController@index');
```

## 5. 添加后台菜单

接下来我们要添加管理后台的菜单，点击左侧的`系统管理`-&gt;`菜单`菜单，然后按下图填写：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/o5FuHJX9aZ.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/o5FuHJX9aZ.png?imageView2/2/w/1240/h/0)

点击`提交`按钮来保存，然后重新排序：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/ta2GP9TqJT.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/ta2GP9TqJT.png?imageView2/2/w/1240/h/0)

点击`保存`按钮，刷新页面，然后点击左侧菜单的`优惠券管理`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/JMDRUHm07L.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/JMDRUHm07L.png!large)

## 6. 生成测试优惠券

接下来我们要编写优惠券的工厂文件来实现测试时优惠券的生成。

在编写工厂文件之前，我们需要先编写优惠码的生成逻辑，我们把这个逻辑放在`CouponCode`模型文件里：

_app/Models/CouponCode.php_

```
use Illuminate\Support\Str;
.
.
.
    public static function findAvailableCode($length = 16)
    {
        do {
            // 生成一个指定长度的随机字符串，并转成大写
            $code = strtoupper(Str::random($length));
        // 如果生成的码已存在就继续循环
        } while (self::query()->where('code', $code)->exists());

        return $code;
    }
```

然后修改对应的工厂文件：

_database/factories/CouponCodeFactory.php_

```
<?php

use App\Models\CouponCode;
use Faker\Generator as Faker;

$factory->define(CouponCode::class, function (Faker $faker) {
    // 首先随机取得一个类型
    $type  = $faker->randomElement(array_keys(CouponCode::$typeMap));
    // 根据取得的类型生成对应折扣
    $value = $type === CouponCode::TYPE_FIXED ? random_int(1, 200) : random_int(1, 50);

    // 如果是固定金额，则最低订单金额必须要比优惠金额高 0.01 元
    if ($type === CouponCode::TYPE_FIXED) {
        $minAmount = $value + 0.01;
    } else {
        // 如果是百分比折扣，有 50% 概率不需要最低订单金额
        if (random_int(0, 100) < 50) {
            $minAmount = 0;
        } else {
            $minAmount = random_int(100, 1000);
        }
    }

    return [
        'name'       => join(' ', $faker->words), // 随机生成名称
        'code'       => CouponCode::findAvailableCode(), // 调用优惠码生成方法
        'type'       => $type,
        'value'      => $value,
        'total'      => 1000,
        'used'       => 0,
        'min_amount' => $minAmount,
        'not_before' => null,
        'not_after'  => null,
        'enabled'    => true,
    ];
});
```

接下来我们在`tinker`中试一下：

```
php artisan tinker
```

```
>>> factory(App\Models\CouponCode::class, 10)->create()
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/lpUcpJCBRz.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/lpUcpJCBRz.png?imageView2/2/w/1240/h/0)

接下来我们去后台看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/jbDc5As7Cx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/jbDc5As7Cx.png!large)

## 7. 优化

可以看到列表字段太多信息比较杂乱，我们来优化一下：`类型`、`折扣`和`最低金额`三个字段可以用更友好的形式输出，比如`满 100 减 50`，`已用`和`总量`可以用`10 / 1000`这样的形式。

`类型`、`折扣`和`最低金额`这三个字段的友好输出可以在多个地方使用，因此可以放到`CouponCode`模型里，作为一个访问器：

_app/Models/CouponCode.php_

```
.
.
.
    protected $appends = ['description'];

    public function getDescriptionAttribute()
    {
        $str = '';

        if ($this->min_amount > 0) {
            $str = '满'.$this->min_amount;
        }
        if ($this->type === self::TYPE_PERCENT) {
            return $str.'优惠'.$this->value.'%';
        }

        return $str.'减'.$this->value;
    }
.
.
.
```

接下来根据我们之前说的优化规则来修改`grid()`方法：

_app/Admin/Controllers/CouponCodesController.php_

```
.
.
.
    protected function grid()
    {
        $grid = new Grid(new CouponCode);

        $grid->model()->orderBy('created_at', 'desc');
        $grid->id('ID')->sortable();
        $grid->name('名称');
        $grid->code('优惠码');
        $grid->description('描述');
        $grid->column('usage', '用量')->display(function ($value) {
            return "{$this->used} / {$this->total}";
        });
        $grid->enabled('是否启用')->display(function ($value) {
            return $value ? '是' : '否';
        });
        $grid->created_at('创建时间');
        $grid->actions(function ($actions) {
            $actions->disableView();
        });

        return $grid;
    }
```

其中`$grid->column('usage', '用量')`是我们虚拟出来的一个字段，然后通过`display()`来输出这个虚拟字段的内容。

然后刷新下页面看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/xGcbzjwqkQ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/xGcbzjwqkQ.png!large)

总体上还不错，但是`描述`那一栏里面有许多不必要的`.00`，应该把这个去掉，优化一下`getDescriptionAttribute()`方法：

_app/Models/CouponCode.php_

```
.
.
.
    public function getDescriptionAttribute()
    {
        $str = '';

        if ($this->min_amount > 0) {
            $str = '满'.str_replace('.00', '', $this->min_amount);
        }
        if ($this->type === self::TYPE_PERCENT) {
            return $str.'优惠'.str_replace('.00', '', $this->value).'%';
        }

        return $str.'减'.str_replace('.00', '', $this->value);
    }
```

再刷新页面看看：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lDSmvPVuAk.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lDSmvPVuAk.png!large)

清爽多了。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 优惠券列表"
```



