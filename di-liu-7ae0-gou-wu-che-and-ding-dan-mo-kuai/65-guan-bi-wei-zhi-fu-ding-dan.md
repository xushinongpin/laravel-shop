## 关闭未支付订单

上一节我们实现了创建订单的功能，在创建订单的同时我们减去了对应商品 SKU 的库存，恶意用户可以通过下大量的订单又不支付来占用商品库存，让正常的用户因为库存不足而无法下单。因此我们需要有一个关闭未支付订单的机制，当创建订单之后一定时间内没有支付，将关闭订单并退回减去的库存。

对于这个需求我们可以用 Laravel 提供的延迟任务（Delayed Job）功能来解决。当我们的系统触发了一个延迟任务时，Laravel 会用当前时间加上任务的延迟时间计算出任务应该被执行的时间戳，然后将这个时间戳和任务信息序列化之后存入队列，Laravel 的队列处理器会不断查询并执行队列中满足预计执行时间等于或早于当前时间的任务。

## 1. 创建任务

我们通过`make:job`命令来创建一个任务：

```
$ php artisan make:job CloseOrder
```

创建的任务类保存在`app/Jobs`目录下，现在编辑刚刚创建的任务类：

_app/Jobs/CloseOrder.php_

```
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use App\Models\Order;

// 代表这个类需要被放到队列中执行，而不是触发时立即执行
class CloseOrder implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $order;

    public function __construct(Order $order, $delay)
    {
        $this->order = $order;
        // 设置延迟的时间，delay() 方法的参数代表多少秒之后执行
        $this->delay($delay);
    }

    // 定义这个任务类具体的执行逻辑
    // 当队列处理器从队列中取出任务时，会调用 handle() 方法
    public function handle()
    {
        // 判断对应的订单是否已经被支付
        // 如果已经支付则不需要关闭订单，直接退出
        if ($this->order->paid_at) {
            return;
        }
        // 通过事务执行 sql
        \DB::transaction(function() {
            // 将订单的 closed 字段标记为 true，即关闭订单
            $this->order->update(['closed' => true]);
            // 循环遍历订单中的商品 SKU，将订单中的数量加回到 SKU 的库存中去
            foreach ($this->order->items as $item) {
                $item->productSku->addStock($item->amount);
            }
        });
    }
}
```

## 2. 触发任务

接下来我们需要在创建订单之后触发这个任务：

_app/Http/Controllers/OrdersController.php_

```
use App\Jobs\CloseOrder;
    .
    .
    .
    public function store(OrderRequest $request)
    {
        .
        .
        .
        $this->dispatch(new CloseOrder($order, config('app.order_ttl')));

        return $order;
    }
```

`CloseOrder`构造函数的第二个参数延迟时间我们从配置文件中读取，**为了方便我们测试，把这个值设置成 30 秒**：

_config/app.php_

```
'order_ttl' => 30,
```

## 3. 测试

默认情况下，Laravel 生成的`.env`文件里把队列的驱动设置成了`sync`（同步），在同步模式下延迟任务会被立即执行，所以需要先把队列的驱动改成`redis`：

_.env_

```
.
.
.
QUEUE_CONNECTION=redis
.
.
.
```

要使用`redis`作为队列驱动，我们还需要引入`predis/predis`这个包

```
$ composer require predis/predis
```

接下来启动队列处理器：

```
$ php artisan queue:work
```

进入商品列表页，任意选择一个商品并将其加入购物车，记住库存数量：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/TIoEqSYAQA.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/TIoEqSYAQA.png!large)

提交订单之后再次进入该商品页面查看库存：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/g0HxZneImz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/g0HxZneImz.png!large)

可以看到数量减少了。

> 一定要在 30 秒内完成这个动作，否则关闭订单的任务就会被执行，库存会被退回。

查看启动队列处理器的终端窗口，可以看到任务已经被执行：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/FWX6q7t9a0.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/FWX6q7t9a0.png!large)

到数据库中查看订单的状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Y6sJwdp86r.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Y6sJwdp86r.png!large)

`closed`字段已经被标成了`1`。

再刷新对应的商品页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/L0zPg64Quo.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/L0zPg64Quo.png!large)

库存回到原本的数值。

## 4. 调整参数

在终端按`ctrl + c`退出队列处理器。

接下来我们把延迟的参数调成`1800`也就是半个小时，如果用户在下单之后半个小时之内没有支付，我们将关闭订单并退回库存。

_config/app.php_

```
'order_ttl' => 1800,
```

> 不同类型的电商网站对这个参数要求不一样，在正式的项目中这个参数通常会由产品经理或者运营根据项目的具体情况决定。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "关闭未支付订单"
```



