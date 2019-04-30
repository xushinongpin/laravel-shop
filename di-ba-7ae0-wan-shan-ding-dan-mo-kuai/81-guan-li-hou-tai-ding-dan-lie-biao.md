[![](https://iocaffcdn.phphub.org/uploads/images/201904/20/5320/DcYMo3gmfo.png!large "管理后台 - 订单列表")](https://iocaffcdn.phphub.org/uploads/images/201904/20/5320/DcYMo3gmfo.png!large)

## 订单列表

接下来我们要在管理后台添加订单管理功能，先实现订单列表。

## 1. 控制器

首先通过`admin:make`命令创建一个管理后台的控制器：

```
$ php artisan admin:make OrdersController --model=App\\Models\\Order
```

需要修改`index()`和`grid()`方法：

_app/Admin/Controllers/OrdersController.php_

```
<?php

namespace App\Admin\Controllers;

use App\Models\Order;
use App\Http\Controllers\Controller;
use Encore\Admin\Controllers\HasResourceActions;
use Encore\Admin\Grid;
use Encore\Admin\Layout\Content;

class OrdersController extends Controller
{
    use HasResourceActions;

    public function index(Content $content)
    {
        return $content
            ->header('订单列表')
            ->body($this->grid());
    }

    protected function grid()
    {
        $grid = new Grid(new Order);

        // 只展示已支付的订单，并且默认按支付时间倒序排序
        $grid->model()->whereNotNull('paid_at')->orderBy('paid_at', 'desc');

        $grid->no('订单流水号');
        // 展示关联关系的字段时，使用 column 方法
        $grid->column('user.name', '买家');
        $grid->total_amount('总金额')->sortable();
        $grid->paid_at('支付时间')->sortable();
        $grid->ship_status('物流')->display(function($value) {
            return Order::$shipStatusMap[$value];
        });
        $grid->refund_status('退款状态')->display(function($value) {
            return Order::$refundStatusMap[$value];
        });
        // 禁用创建按钮，后台不需要创建订单
        $grid->disableCreateButton();
        $grid->actions(function ($actions) {
            // 禁用删除和编辑按钮
            $actions->disableDelete();
            $actions->disableEdit();
        });
        $grid->tools(function ($tools) {
            // 禁用批量删除按钮
            $tools->batch(function ($batch) {
                $batch->disableDelete();
            });
        });

        return $grid;
    }
}
```

## 2. 路由

接下来添加对应的路由：

_app/Admin/routes.php_

```
$router->get('orders', 'OrdersController@index')->name('admin.orders.index');
```

## 3. 添加菜单

点击左侧菜单的`系统管理`-&gt;`菜单`，拉到页面底部，参照下图填空：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/3nRFdnVDUp.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/3nRFdnVDUp.png?imageView2/2/w/1240/h/0)

点击`提交`按钮，然后将`订单管理`拖到`商品管理`下方，并点击`保存`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/f4JDfSuRF0.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/f4JDfSuRF0.png?imageView2/2/w/1240/h/0)

刷新页面之后在左侧即可看到刚刚添加的菜单了：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/uuJuGpyjHH.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/uuJuGpyjHH.png?imageView2/2/w/1240/h/0)

点进去看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/20/5320/DcYMo3gmfo.png!large "管理后台 - 订单列表")](https://iocaffcdn.phphub.org/uploads/images/201904/20/5320/DcYMo3gmfo.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 订单列表"
```



