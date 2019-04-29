## 商品模型

本章节将要实现管理后台的商品列表，为下一节的商品创建做准备。

## 1. 创建控制器

用`admin:make`来创建管理后台的控制器：

```
$ php artisan admin:make ProductsController --model=App\\Models\\Product
```

本章节要解决商品列表的展示，因此我们先调整`index()`和`grid()`方法：

_app/Admin/Controllers/ProductsController.php_

```
  .
    .
    .
    public function index(Content $content)
    {
        return $content
            ->header('商品列表')
            ->body($this->grid());
    }
    .
    .
    .
    protected function grid()
    {
        $grid = new Grid(new Product);

        $grid->id('ID')->sortable();
        $grid->title('商品名称');
        $grid->on_sale('已上架')->display(function ($value) {
            return $value ? '是' : '否';
        });
        $grid->price('价格');
        $grid->rating('评分');
        $grid->sold_count('销量');
        $grid->review_count('评论数');

        $grid->actions(function ($actions) {
            $actions->disableView();
            $actions->disableDelete();
        });
        $grid->tools(function ($tools) {
            // 禁用批量删除按钮
            $tools->batch(function ($batch) {
                $batch->disableDelete();
            });
        });

        return $grid;
    }
    .
    .
    .
```

每一个`$grid->`调用，对应后台表格里的一个列显示，这个代码和之前的《用户列表》章节十分类似，这里就不一一解释了。

然后配置一下路由：

_app/Admin/routes.php_

```
$router->get('products', 'ProductsController@index');
```

## 2. 添加菜单

接下来我们在管理后台添加一个商品管理的菜单：

访问[http://shop.test/admin/](http://shop.test/admin/)，点击左侧菜单`系统管理`-&gt;`菜单`，按照下图填入各个字段：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/12/5320/foGZbI1EjU.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/12/5320/foGZbI1EjU.png?imageView2/2/w/1240/h/0)

点击`提交`按钮，然后把`商品管理`拖到`用户管理`下方并点击`保存`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/drxZmu8B3u.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/drxZmu8B3u.png?imageView2/2/w/1240/h/0)

刷新浏览器，就可以在左侧看到刚刚添加的`商品管理`菜单了：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/CMCU2tIgSc.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/CMCU2tIgSc.png?imageView2/2/w/1240/h/0)

点进去看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/Wmh6BPIQHy.png!large "后台商品列表")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/Wmh6BPIQHy.png!large)

## Git 代码版本控制

接着让我们将这些变更加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 商品列表"
```



