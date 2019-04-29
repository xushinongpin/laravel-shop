## 创建商品

我们开发的是 B2C 商城，可以理解为『互联网超市』。管理员在后台录入商品、品种和价格，用户在网站页面上购买。接下来我们要实现后台创建商品页面以及处理创建商品逻辑。

## 1. 修改控制器

修改`ProductsController`的`create()`和`form()`方法：

_app/Admin/Controllers/ProductsController.php_

```
   .
    .
    .
    public function create(Content $content)
    {
        return $content
            ->header('创建商品')
            ->body($this->form());
    }

    protected function form()
    {
        $form = new Form(new Product);

        // 创建一个输入框，第一个参数 title 是模型的字段名，第二个参数是该字段描述
        $form->text('title', '商品名称')->rules('required');

        // 创建一个选择图片的框
        $form->image('image', '封面图片')->rules('required|image');

        // 创建一个富文本编辑器
        $form->editor('description', '商品描述')->rules('required');

        // 创建一组单选框
        $form->radio('on_sale', '上架')->options(['1' => '是', '0'=> '否'])->default('0');

        // 直接添加一对多的关联模型
        $form->hasMany('skus', 'SKU 列表', function (Form\NestedForm $form) {
            $form->text('title', 'SKU 名称')->rules('required');
            $form->text('description', 'SKU 描述')->rules('required');
            $form->text('price', '单价')->rules('required|numeric|min:0.01');
            $form->text('stock', '剩余库存')->rules('required|integer|min:0');
        });

        // 定义事件回调，当模型即将保存时会触发这个回调
        $form->saving(function (Form $form) {
            $form->model()->price = collect($form->input('skus'))->where(Form::REMOVE_FLAG_NAME, 0)->min('price') ?: 0;
        });

        return $form;
    }
```

代码解析：

* 通过
  `rules('required')`
  方法可以定义对应字段在提交时的校验规则，验证规则与 Laravel 的验证规则一致。
* `$form-`
  `>`
  `radio('on_sale', '上架')`
  在表单中创建一组单选框，
  `options(['1' =`
  `>`
  ` '是', '0'=`
  `>`
  ` '否'])`
  设置两个选项，
  `default('0')`
  代表默认选择值为
  `0`
  的框，在我们这里就是默认为
  `否`
  。
* `$form-`
  `>`
  `hasMany('skus', 'SKU 列表', /**/)`
  可以在表单中直接添加一对多的关联模型，商品和商品 SKU 的关系就是一对多，第一个参数必须和主模型中定义此关联关系的方法同名，我们之前在
  `App\Models\Product`
  类中定义了
  `skus()`
  方法来关联 SKU，因此这里我们需要填入
  `skus`
  ，第二个参数是对这个关联关系的描述，第三个参数是一个匿名函数，用来定义关联模型的字段。
* `$form-`
  `>`
  `saving()`
  用来定义一个事件回调，当模型即将保存时会触发这个回调。我们需要在保存商品之前拿到所有 SKU 中最低的价格作为商品的价格，然后通过
  `$form-`
  `>`
  `model()-`
  `>`
  `price`
  存入到商品模型中。
* `collect()`
  函数是 Laravel 提供的一个辅助函数，可以快速创建一个
  `Collection`
  对象。在这里我们把用户提交上来的 SKU 数据放到
  `Collection`
  中，利用
  `Collection`
  提供的
  `min()`
  方法求出所有 SKU 中最小的
  `price`
  ，后面的
  `?: 0`
  则是保证当 SKU 数据为空时
  `price`
  字段被赋值
  `0`
  。

> `where(Form::REMOVE_FLAG_NAME, 0)`这个代码的详细解释可以查看这篇帖子：[https://learnku.com/laravel/t/13432/once-the-product-is-created-edit-it-again-find-the-lowest-price-sku-directly-and-save-it-the-price-of-the-commodity-database-will-not-be-updated-how-can-i-solve-it-thank-you](https://learnku.com/laravel/t/13432/once-the-product-is-created-edit-it-again-find-the-lowest-price-sku-directly-and-save-it-the-price-of-the-commodity-database-will-not-be-updated-how-can-i-solve-it-thank-you)

## 2. 添加路由

_app/Admin/routes.php_

```
    $router->get('products/create', 'ProductsController@create');
    $router->post('products', 'ProductsController@store');
```

在`ProductsController`中并没有`store`方法，这是因为 Laravel-Admin 在创建控制器的时候默认引入了`HasResourceActions`这个 Trait，打开`Encore\Admin\Controllers\HasResourceActions`这个类可以看到里面定义了`store`方法：

```
public function store()
{
    return $this->form()->store();
}
```

## 3. 测试添加商品

访问管理后台，进入商品管理页面，点击`新增`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/rYBe8TDf4Q.png!large "后台创建和编辑商品")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/rYBe8TDf4Q.png!large)

会看到有错误提示：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/pnu4jNet9o.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/pnu4jNet9o.png!large)

这是因为 Laravel-Admin 为了避免加载太多前端静态文件，默认禁用了`editor`这个表单组件，我们可以在`app/Admin/bootstrap.php`中把这个禁用解除：

_app/Admin/bootstrap.php_

把

```
Encore\Admin\Form::forget(['map', 'editor']);
```

修改为：

```
Encore\Admin\Form::forget(['map']);
```

刷新页面，然后再次点击`新增`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/SBhSt62f8l.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/SBhSt62f8l.png!large)

填入一些测试数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/SlUXCT6rtR.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/SlUXCT6rtR.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/Ff9bQb5dHv.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/Ff9bQb5dHv.png!large)

点击`SKU 列表`下的`新增`按钮来新建 SKU：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/YUBRFPbuQy.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/YUBRFPbuQy.png?imageView2/2/w/1240/h/0)

然后点击最下方的`提交`按钮，可以看到创建成功并自动跳转到商品列表页：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/ObZ86zMfOz.png!large "后台创建和编辑商品")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/ObZ86zMfOz.png!large)

显示的价格也是我们填入的较小的那一个，说明我们的逻辑没有问题。

## 4. 编辑商品

接下来我们要实现编辑商品，只需要稍微改动一下`ProductsController`的`edit()`方法即可：

_app/Admin/Controllers/ProductsController.php_

```
.
.
.
    public function edit($id, Content $content)
    {
        return $content
            ->header('编辑商品')
            ->body($this->form()->edit($id));
    }
```

然后配上路由

_app/Admin/routes.php_

```
$router->get('products/{id}/edit', 'ProductsController@edit');
$router->put('products/{id}', 'ProductsController@update');
```

同样的，控制器中的`update()`方法也是来自`HasResourceActions`这个 Trait：

```
public function update($id)
{
    return $this->form()->update($id);
}
```

在商品列表页面，点击刚刚我们创建的商品最右边的编辑按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/Sgf2RvUVUZ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/Sgf2RvUVUZ.png!large)

可以看到别的数据都正常，但是图片显示不出来：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/w00CUjyqNz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/w00CUjyqNz.png!large)

调出 Chrome 的开发者工具，发现是图片 404 了：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/blE9vwm18i.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/blE9vwm18i.png!large)

这是因为我们上传文件的都是存储在`storage`目录下，而 HTTP 服务器指向的根目录是`public`目录，要想用户能通过浏览器访问`storage`目录下的文件，需要创建一个软链接，Laravel 内置了这个命令：

```
$ php artisan storage:link
```

再刷新一下页面，可以看到图片展示正常：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/pqbIp8Y9oS.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/pqbIp8Y9oS.png!large)

接下来我们试着修改部分字段：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/DKrqWYgtMe.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/DKrqWYgtMe.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/rTRNRgmZel.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/rTRNRgmZel.png!large)

然后点击最下方的`提交`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/bHVu5qOwZu.png!large "后台创建和编辑商品")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/bHVu5qOwZu.png!large)

可以看到修改成功。

## 5. 移除无用方法

由于我们不需要在后台展示商品详情页，因此可以删掉控制器中的`show()`和`detail()`方法。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 新增、编辑商品"
```



