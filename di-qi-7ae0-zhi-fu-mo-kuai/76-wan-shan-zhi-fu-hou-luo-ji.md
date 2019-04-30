## 支付后逻辑

前两个章节我们已经完成了订单支付的部分，接下来我们还要补充一些逻辑，比如支付之后要给订单中的商品增加销量，比如我们要发邮件给用户告知订单支付成功。

商品增加销量和发送邮件并不会影响到订单的支付状态，即使这两个操作失败了也不影响我们后续的业务流程，对于此类需求我们通常使用异步事件来解决。

## 支付成功事件

接下来我们来创建一个支付成功的事件：

```
$ php artisan make:event OrderPaid
```

_app/Events/OrderPaid.php_

```
use App\Models\Order;
.
.
.
class OrderPaid
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    protected $order;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    public function getOrder()
    {
        return $this->order;
    }
}
```

事件本身不需要有逻辑，只需要包含相关的信息即可，在我们这个场景里就只需要一个订单对象。

接下来我们在支付成功的服务器端回调里触发这个事件：

_app/Http/Controllers/PaymentController.php_

```
use App\Events\OrderPaid;
.
.
.
    public function alipayNotify()
    {
        .
        .
        .
        $this->afterPaid($order);

        return app('alipay')->success();
    }

    public function wechatNotify()
    {
        .
        .
        .
        $this->afterPaid($order);

        return app('wechat_pay')->success();
    }

    protected function afterPaid(Order $order)
    {
        event(new OrderPaid($order));
    }
```

接下来我们就要来实现支付完成事件对应的监听器。

## 商品增加销量

访问商品列表页，可以看到现在`nam`这个商品的销量还是`0`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/pFukEOyzku.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/pFukEOyzku.png!large)

### 1. 创建监听器

我们希望订单支付之后对应的商品销量会对应地增加，所以创建一个更新商品销量的监听器：

```
$ php artisan make:listener UpdateProductSoldCount --event=OrderPaid
```

_app/Listeners/UpdateProductSoldCount.php_

```
<?php

namespace App\Listeners;

use App\Events\OrderPaid;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;
use App\Models\OrderItem;
//  implements ShouldQueue 代表此监听器是异步执行的
class UpdateProductSoldCount implements ShouldQueue
{
    // Laravel 会默认执行监听器的 handle 方法，触发的事件会作为 handle 方法的参数
    public function handle(OrderPaid $event)
    {
        // 从事件对象中取出对应的订单
        $order = $event->getOrder();
        // 预加载商品数据
        $order->load('items.product');
        // 循环遍历订单的商品
        foreach ($order->items as $item) {
            $product   = $item->product;
            // 计算对应商品的销量
            $soldCount = OrderItem::query()
                ->where('product_id', $product->id)
                ->whereHas('order', function ($query) {
                    $query->whereNotNull('paid_at');  // 关联的订单状态是已支付
                })->sum('amount');
            // 更新商品销量
            $product->update([
                'sold_count' => $soldCount,
            ]);
        }
    }
}
```

### 2. 关联事件和监听器

别忘了在`EventServiceProvider`中将事件和监听器关联起来：

_app/Providers/EventServiceProvider.php_

```
use App\Events\OrderPaid;
use App\Listeners\UpdateProductSoldCount;
.
.
.
    protected $listen = [
        .
        .
        .
        OrderPaid::class => [
            UpdateProductSoldCount::class,
        ],
    ];
```

## 发送通知邮件

接下来我们要实现订单支付成功时给用户发送通知邮件。

### 1. 创建通知类

这里我们通过 Laravel 内置的消息通知系统（Notification）来发送邮件，使用`make:notification`命令来创建一个新的通知：

```
$ php artisan make:notification OrderPaidNotification
```

_app/Notifications/OrderPaidNotification.php_

```
<?php

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Notification;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use App\Models\Order;

class OrderPaidNotification extends Notification
{
    use Queueable;

    protected $order;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    // 我们只需要通过邮件通知，因此这里只需要一个 mail 即可
    public function via($notifiable)
    {
        return ['mail'];
    }

    public function toMail($notifiable)
    {
        return (new MailMessage)
                    ->subject('订单支付成功')  // 邮件标题
                    ->greeting($this->order->user->name.'您好：') // 欢迎词
                    ->line('您于 '.$this->order->created_at->format('m-d H:i').' 创建的订单已经支付成功。') // 邮件内容
                    ->action('查看订单', route('orders.show', [$this->order->id])) // 邮件中的按钮及对应链接
                    ->success(); // 按钮的色调
    }
}
```

### 2. 创建监听器

接下来创建监听器来执行发送邮件的动作：

```
$ php artisan make:listener SendOrderPaidMail --event=OrderPaid
```

_app/Listeners/SendOrderPaidMail.php_

```
<?php

namespace App\Listeners;

use App\Events\OrderPaid;
use App\Notifications\OrderPaidNotification;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;

// implements ShouldQueue 代表异步监听器
class SendOrderPaidMail implements ShouldQueue
{
    public function handle(OrderPaid $event)
    {
        // 从事件对象中取出对应的订单
        $order = $event->getOrder();
        // 调用 notify 方法来发送通知
        $order->user->notify(new OrderPaidNotification($order));
    }
}
```

### 3. 关联事件和监听器

同样在`EventServiceProvider`中将事件和监听器关联起来：

_app/Providers/EventServiceProvider.php_

```
use App\Listeners\SendOrderPaidMail;
.
.
.
    protected $listen = [
        .
        .
        .
        OrderPaid::class => [
            UpdateProductSoldCount::class,
            SendOrderPaidMail::class,
        ],
    ];
```

## 测试

由于我们定义的事件监听器都是异步的，因此在测试之前需要先启动队列处理器：

```
$ php artisan queue:work
```

从数据库中找到一条已经支付成功的订单并记录下其 ID：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/2zBaaSNNXU.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/2zBaaSNNXU.png?imageView2/2/w/1240/h/0)

然后在终端里进入 tinker：

```
$ php artisan tinker
```

在 tinker 中触发订单支付成功的事件，事件对应的订单就是我们刚刚在数据库中找的那一条：

```
>>> event(new App\Events\OrderPaid(App\Models\Order::find(17)))
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/eMdC7Imrm4.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/04/5320/eMdC7Imrm4.png?imageView2/2/w/1240/h/0)

这个时候看到启动队列处理的窗口有了输出：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/1UkRZuFHRV.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/1UkRZuFHRV.png!large)

再进入到商品列表页看看销量是否变化：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/R88GdSG86Z.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/R88GdSG86Z.png!large)

访问[http://shop.test:8025/](http://shop.test:8025/)进入 MailHog

[![](https://iocaffcdn.phphub.org/uploads/images/201805/26/5320/BLZ6Dui05G.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/26/5320/BLZ6Dui05G.png?imageView2/2/w/1240/h/0)

[![](https://iocaffcdn.phphub.org/uploads/images/201805/26/5320/bOKl05i3Ew.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/26/5320/bOKl05i3Ew.png?imageView2/2/w/1240/h/0)

可以看到收到邮件。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "完善支付后逻辑"
```



