## 收藏商品

收藏商品是电商网站一个常用的功能，本章节要实现收藏商品的基本功能。

## 1. 数据库结构

收藏商品本质上是用户和商品的多对多关联，因此不需要创建新的模型，只需要增加一个中间表即可：

```
$ php artisan make:migration create_user_favorite_products_table --create=user_favorite_products
```

> 注意：中间表命名，要越直白越好，名字长点也无所谓。一个简单的判断命名是否合格的方法是 —— 想象自己半年一年以后是否能快速地从数据库表名得知此表的功能。

_database/migrations/&lt; your\_date &gt;\_create\_user\_favorite\_products\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('user_favorite_products', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('user_id');
            $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
            $table->unsignedBigInteger('product_id');
            $table->foreign('product_id')->references('id')->on('products')->onDelete('cascade');
            $table->timestamps();
        });
    }
.
.
.
```

> 这里我们保留了时间戳的字段，这是因为我们希望用户看到的收藏商品是按用户收藏时间排序的，所以需要有一个时间字段用于排序。

然后执行数据库迁移：

```
$ php artisan migrate
```

## 2. 模型文件增加关联

接下来我们在`User`模型中增加与商品的关联关系：

_app/Models/User.php_

```
.
.
.
    public function favoriteProducts()
    {
        return $this->belongsToMany(Product::class, 'user_favorite_products')
            ->withTimestamps()
            ->orderBy('user_favorite_products.created_at', 'desc');
    }
.
.
.
```

代码解析：

* `belongsToMany()`
  方法用于定义一个多对多的关联，第一个参数是关联的模型类名，第二个参数是中间表的表名。
* `withTimestamps()`
  代表中间表带有时间戳字段。
* `orderBy('user_favorite_products.created_at', 'desc')`
  代表默认的排序方式是根据中间表的创建时间倒序排序。

## 3. 控制器

接下来我们需要在`ProductsController`中新增收藏的接口：

_app/Http/Controllers/ProductsController.php_

```
.
.
.
    public function favor(Product $product, Request $request)
    {
        $user = $request->user();
        if ($user->favoriteProducts()->find($product->id)) {
            return [];
        }

        $user->favoriteProducts()->attach($product);

        return [];
    }
.
.
.
```

这段代码先是判断当前用户是否已经收藏了此商品，如果已经收藏则不做任何操作直接返回，否则通过`attach()`方法将当前用户和此商品关联起来。

> `attach()`方法的参数可以是模型的 id，也可以是模型对象本身，因此这里还可以写成`attach($product->id)`。

然后是取消收藏的接口：

_app/Http/Controllers/ProductsController.php_

```
.
.
.
    public function disfavor(Product $product, Request $request)
    {
        $user = $request->user();
        $user->favoriteProducts()->detach($product);

        return [];
    }
.
.
.
```

`detach()`方法用于取消多对多的关联，接受的参数个数与`attach()`方法一致。

## 4. 路由

接下来把这两个接口添加到路由中：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
.
.
.
    Route::post('products/{product}/favorite', 'ProductsController@favor')->name('products.favor');
    Route::delete('products/{product}/favorite', 'ProductsController@disfavor')->name('products.disfavor');
});
```

## 5. 『收藏』按钮

接下来我们需要在前端模板页面实现『收藏』按钮的功能：

_resources/views/products/show.blade.php_

```
.
.
.
  $(document).ready(function () {
    .
    .
    .
    // 监听收藏按钮的点击事件
    $('.btn-favor').click(function () {
      // 发起一个 post ajax 请求，请求 url 通过后端的 route() 函数生成。
      axios.post('{{ route('products.favor', ['product' => $product->id]) }}')
        .then(function () { // 请求成功会执行这个回调
          swal('操作成功', '', 'success');
        }, function(error) { // 请求失败会执行这个回调
          // 如果返回码是 401 代表没登录
          if (error.response && error.response.status === 401) {
            swal('请先登录', '', 'error');
          } else if (error.response && error.response.data.msg) {
            // 其他有 msg 字段的情况，将 msg 提示给用户
            swal(error.response.data.msg, '', 'error');
          }  else {
            // 其他情况应该是系统挂了
            swal('系统错误', '', 'error');
          }
        });
    });

  });
</script>
@endsection
```

现在来看看效果，首先确保**没有登录**，然后进入任意一个商品详情页，点击`收藏`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/0GxWLrI1Ao.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/0GxWLrI1Ao.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/ovzXS6jKIL.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/ovzXS6jKIL.png!large)

然后登陆之后再点击看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/7O7uYoNvza.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/7O7uYoNvza.png!large)

再到数据库中看看是否已经有对应的关联记录：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/ki0tOEBioH.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/ki0tOEBioH.png!large)

## 6. 『取消收藏』 按钮

接下来我们要在页面添加`取消收藏`按钮及其功能，对于已经收藏了当前商品的用户，我们不展示`加入收藏`按钮，而展示`取消收藏`按钮，因此需要在控制器中把收藏状态注入到模板中：

_app/Http/Controllers/ProductsController.php_

```
.
.
.
    public function show(Product $product, Request $request)
    {
        if (!$product->on_sale) {
            throw new InvalidRequestException('商品未上架');
        }

        $favored = false;
        // 用户未登录时返回的是 null，已登录时返回的是对应的用户对象
        if($user = $request->user()) {
            // 从当前用户已收藏的商品中搜索 id 为当前商品 id 的商品
            // boolval() 函数用于把值转为布尔值
            $favored = boolval($user->favoriteProducts()->find($product->id));
        }

        return view('products.show', ['product' => $product, 'favored' => $favored]);
    }
.
.
.
```

有了`$favored`这个变量之后，我们在模板里加上判断：

_resources/views/products/show.blade.php_

```
.
.
.
        <div class="buttons">
          @if($favored)
            <button class="btn btn-danger btn-disfavor">取消收藏</button>
          @else
            <button class="btn btn-success btn-favor">❤ 收藏</button>
          @endif
          <button class="btn btn-primary btn-add-to-cart">加入购物车</button>
        </div>
.
.
.
  $(document).ready(function () {
    .
    .
    .
    $('.btn-favor').click(function () {
      axios.post('{{ route('products.favor', ['product' => $product->id]) }}')
        .then(function () {
          swal('操作成功', '', 'success')
          .then(function () {  // 这里加了一个 then() 方法
              location.reload();
            });
        }, function(error) {
          if (error.response && error.response.status === 401) {
            swal('请先登录', '', 'error');
          }  else if (error.response && error.response.data.msg) {
            swal(error.response.data.msg, '', 'error');
          }  else {
            swal('系统错误', '', 'error');
          }
        });
    });

    $('.btn-disfavor').click(function () {
      axios.delete('{{ route('products.disfavor', ['product' => $product->id]) }}')
        .then(function () {
          swal('操作成功', '', 'success')
            .then(function () {
              location.reload();
            });
        });
    });

  })
```

> 我们在`swal()`的回调里调用了`location.reload()`刷新页面来刷新收藏按钮的状态，当用户点击弹出框的`OK`按钮时这个回调会被触发。

刷新页面看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/RWO6uVb62J.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/RWO6uVb62J.gif!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 商品收藏"
```



