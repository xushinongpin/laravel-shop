[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/pnuBT4Wgl5.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/pnuBT4Wgl5.png?imageView2/2/w/1240/h/0)

## 商品列表

本章节将要实现商品列表在用户端的展示。

## 1. 创建控制器

通过`make:controller`创建`ProductsController`：

```
$ php artisan make:controller ProductsController
```

在`ProductsController`中添加`index()`方法：

_app/Http/Controllers/ProductsController.php_

```
use App\Models\Product;
.
.
.
    public function index(Request $request)
    {
        $products = Product::query()->where('on_sale', true)->paginate();

        return view('products.index', ['products' => $products]);
    }
```

代码解析：

* `where('on_sale', true)`
  筛选出
  `on_sale`
  字段为
  `true`
  的记录，这样未上架的商品就不会被展示出来。
* `paginate()`
  分页取出数据。

## 2. 前端模板

创建对应的前端模板文件：

```
$ mkdir -p resources/views/products && touch resources/views/products/index.blade.php
```

_resources/views/products/index.blade.php_

```
@extends('layouts.app')
@section('title', '商品列表')

@section('content')
<div class="row">
<div class="col-lg-10 offset-lg-1">
<div class="card">
  <div class="card-body">
    <div class="row products-list">
      @foreach($products as $product)
        <div class="col-3 product-item">
          <div class="product-content">
            <div class="top">
              <div class="img"><img src="{{ $product->image }}" alt=""></div>
              <div class="price"><b>￥</b>{{ $product->price }}</div>
              <div class="title">{{ $product->title }}</div>
            </div>
            <div class="bottom">
              <div class="sold_count">销量 <span>{{ $product->sold_count }}笔</span></div>
              <div class="review_count">评价 <span>{{ $product->review_count }}</span></div>
            </div>
          </div>
        </div>
      @endforeach
    </div>
  </div>
</div>
</div>
</div>
@endsection
```

美化样式：

_resources/sass/app.scss_

```
.
.
.
.products-index-page {
  .products-list {
    padding: 0 15px;
    .product-item {
      padding: 0 5px;
      margin-bottom: 10px;
      .product-content {
        border: 1px solid #eee;
        .top {
          padding: 5px;
          img {
            width: 100%;
          }
          .price {
            margin-top: 5px;
            font-size: 20px;
            color: #ff0036;
            b {
              font-size: 14px;
            }
          }
          .title {
            margin-top: 10px;
            height: 32px;
            line-height: 12px;
            max-height: 32px;
            a {
              font-size: 12px;
              line-height: 14px;
              color: #333;
              text-decoration: none;
            }
          }
        }
        .bottom {
          font-size: 12px;
          display: flex;
          .sold_count span {
            color: #b57c5b;
            font-weight: bold;
          }
          .review_count span {
            color: #38b;
            font-weight: bold;
          }
          &>div {
            &:first-child {
              border-right: 1px solid #eee;
            }
            padding: 10px 5px;
            line-height: 12px;
            flex-grow: 1;
            border-top: 1px solid #eee;
          }
        }
      }
    }
  }
}
```

## 3. 配置路由

我们希望用户一进来就能看到商品列表，因此让首页直接跳转到商品页面，记得删除原有的首页路由：

_routes/web.php_

```
Route::redirect('/', '/products')->name('root');
Route::get('products', 'ProductsController@index')->name('products.index');
```

> 我们希望游客也能够访问商品列表，所以这条路由不需要放到带有`auth`中间件的路由组中。

## 4. 查看效果

现在直接访问[http://shop.test/](http://shop.test/)，可以看到跳转到了商品列表页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/uXMu51AwmN.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/uXMu51AwmN.png?imageView2/2/w/1240/h/0)

由于我们之前创建的商品没有设置成上架，因此现在列表是空的，我们到后台上架商品：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/ycwvrk6DHY.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/ycwvrk6DHY.png?imageView2/2/w/1240/h/0)

保存之后再次访问商品页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/ZS9AfKDbQz.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/ZS9AfKDbQz.png?imageView2/2/w/1240/h/0)

发现图片没有正常展示，这是因为商品的`image`字段保存的是图片的相对于目录`storage/app/public/`的路径，需要转成绝对路径才能正常展示，我们可以给商品模型加一个访问器来输出绝对路径：

_app/Models/Product.php_

```
use Illuminate\Support\Str;
.
.
.
    public function getImageUrlAttribute()
    {
        // 如果 image 字段本身就已经是完整的 url 就直接返回
        if (Str::startsWith($this->attributes['image'], ['http://', 'https://'])) {
            return $this->attributes['image'];
        }
        return \Storage::disk('public')->url($this->attributes['image']);
    }
```

这里`\Storage::disk('public')`的参数`public`需要和我们在`config/admin.php`里面的`upload.disk`配置一致。

然后修改模板文件，改为输出我们刚刚加上的访问器：

_resources/views/products/index.blade.php_

```
<div class="img"><img src="{{ $product->image_url }}" alt=""></div>
```

> Laravel 的模型访问器会自动把下划线改为驼峰，所以`image_url`对应的就是`getImageUrlAttribute`。

再次刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/Txy8Lvblv7.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/Txy8Lvblv7.png?imageView2/2/w/1240/h/0)

## 5. 生成测试数据

接下来我们要生成一批测试商品，先写商品的工厂文件：

_database/factories/ProductFactory.php_

```
<?php

use App\Models\Product;
use Faker\Generator as Faker;

$factory->define(Product::class, function (Faker $faker) {
    $image = $faker->randomElement([
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/7kG1HekGK6.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/1B3n0ATKrn.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/r3BNRe4zXG.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/C0bVuKB2nt.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/82Wf2sg8gM.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/nIvBAQO5Pj.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/XrtIwzrxj7.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/uYEHCJ1oRp.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/2JMRaFwRpo.jpg",
        "https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/pa7DrV43Mw.jpg",
    ]);

    return [
        'title'        => $faker->word,
        'description'  => $faker->sentence,
        'image'        => $image,
        'on_sale'      => true,
        'rating'       => $faker->numberBetween(0, 5),
        'sold_count'   => 0,
        'review_count' => 0,
        'price'        => 0,
    ];
});
```

其中`image`字段的值从一批已经上传好的封面图里随机取得。

接下来是 SKU 的工厂文件：

_database/factories/ProductSkuFactory.php_

```
<?php

use App\Models\ProductSku;
use Faker\Generator as Faker;

$factory->define(ProductSku::class, function (Faker $faker) {
    return [
        'title'       => $faker->word,
        'description' => $faker->sentence,
        'price'       => $faker->randomNumber(4),
        'stock'       => $faker->randomNumber(5),
    ];
});
```

然后我们创建一个 Seeder 文件用来批量创建商品以及对应的 SKU：

```
$ php artisan make:seeder ProductsSeeder
```

_database/seeds/ProductsSeeder.php_

    .
    .
    .
        public function run()
        {
            // 创建 30 个商品
            $products = factory(\App\Models\Product::class, 30)->create();
            foreach ($products as $product) {
                // 创建 3 个 SKU，并且每个 SKU 的 `product_id` 字段都设为当前循环的商品 id
                $skus = factory(\App\Models\ProductSku::class, 3)->create(['product_id' => $product->id]);
                // 找出价格最低的 SKU 价格，把商品价格设置为该价格
                $product->update(['price' => $skus->min('price')]);
            }
        }

最后我们执行这个 Seeder：

```
$ php artisan db:seed --class=ProductsSeeder
```

刷新页面看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/pnuBT4Wgl5.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/pnuBT4Wgl5.png?imageView2/2/w/1240/h/0)

> 商品列表页面要求所有的商品封面图的尺寸比例一致才会美观，如果之前自己上传的封面图片长宽比例不是 4 : 3，那么列表页就会有些许的变形，由于我们之前上传的 iPhone X 图片是正方形的，因此需要在后台将其下架隐藏。

## 6. 完善前端页面

拉到页面底部的时候发现右下角缺了一个，不太美观：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/skVqHb5492.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/skVqHb5492.png?imageView2/2/w/1240/h/0)

默认情况下`paginate()`方法是 15 个每页，而我们的页面是 4 个一行，所以需要调整一下每页展示的商品个数，改成 16 个每页：

_app/Http/Controllers/ProductsController.php_

```
.
.
.
$products = Product::query()->where('on_sale', true)->paginate(16);
.
.
.
```

接下来需要在页面底部加上分页组件：

_resources/views/products/index.blade.php_

```
.
.
.
    <div class="row products-list">
      @foreach($products as $product)
      .
      .
      .
      @endforeach
    </div>
    <div class="float-right">{{ $products->render() }}</div>  <!-- 只需要添加这一行 -->
.
.
.
```

刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/f0Xq9Tqbkt.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/f0Xq9Tqbkt.png?imageView2/2/w/1240/h/0)

分页组件与商品列表有些许的不对齐，调整一下：

_resources/sass/app.scss_

```
.products-index-page {
  .pagination {
    margin-right: 5px;
  }
  .
  .
  .
}
```

再刷新：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/iQPo5fOH1V.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/05/5320/iQPo5fOH1V.png?imageView2/2/w/1240/h/0)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 商品列表"
```



