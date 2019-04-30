## 收藏商品列表

上一节我们实现了收藏商品的功能，接下来本章节要实现收藏商品的列表页面。

## 1. 控制器

在`ProductsController`中添加一个`favorites()`方法：

_app/Http/Controllers/ProductsController.php_

```
.
.
.
    public function favorites(Request $request)
    {
        $products = $request->user()->favoriteProducts()->paginate(16);

        return view('products.favorites', ['products' => $products]);
    }
```

这里我们用分页的方式取出当前用户的收藏商品，由于我们在定义关联关系的时候就已经加上了排序规则，这里就不需要再次设置了。

## 2. 前端模板

创建一个新的模板文件：

```
$ touch resources/views/products/favorites.blade.php
```

_resources/views/products/favorites.blade.php_

```
@extends('layouts.app')
@section('title', '我的收藏')

@section('content')
<div class="row">
<div class="col-lg-10 offset-lg-1">
  <div class="card">
    <div class="card-header">我的收藏</div>
    <div class="card-body">
      <div class="row products-list">
        @foreach($products as $product)
          <div class="col-3 product-item">
            <div class="product-content">
              <div class="top">
                <div class="img">
                  <a href="{{ route('products.show', ['product' => $product->id]) }}">
                    <img src="{{ $product->image_url }}" alt="">
                  </a>
                </div>
                <div class="price"><b>￥</b>{{ $product->price }}</div>
                <a href="{{ route('products.show', ['product' => $product->id]) }}">{{ $product->title }}</a>
              </div>
              <div class="bottom">
                <div class="sold_count">销量 <span>{{ $product->sold_count }}笔</span></div>
                <div class="review_count">评价 <span>{{ $product->review_count }}</span></div>
              </div>
            </div>
          </div>
        @endforeach
      </div>
      <div class="float-right">{{ $products->render() }}</div>
    </div>
  </div>
</div>
</div>
@endsection
```

代码基本上与商品列表页一致。

## 3. 路由

接下来添加路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::get('products/favorites', 'ProductsController@favorites')->name('products.favorites');
});
```

现在测试一下：访问[http://shop.test/products/favorites](http://shop.test/products/favorites)

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/tP5Q5iQ6G6.png!large "收藏商品列表")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/tP5Q5iQ6G6.png!large)

明明已经添加好路由却提示页面不存在，仔细观察 URL，发现和之前的`products/{product}`这个路由冲突了，Laravel 在匹配路由的时候会按定义的顺序依次查找，找到第一个匹配的路由就返回。所以当我们访问这个 URL 的时候会先匹配到商品详情页这个路由，然后把`favorites`当成商品 ID 去数据库查找，查不到对应的商品就抛出了不存在的异常。

解决方案也很简单，把`products/{product}`这个路由放到后面即可：

_routes/web.php_

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/lComWwa2hs.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/lComWwa2hs.png!large)

再次刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/omFoRH9zmd.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/omFoRH9zmd.png!large)

发现样式不对，我们完全可以复用之前商品列表的样式：

_resources/sass/app.scss_

```
.products-index-page, .products-favorites-page {
.
.
.
}
```

再次刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/zCugWPSPlF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/zCugWPSPlF.png!large)

## 4. 添加入口

最后我们需要在页面菜单里添加`我的收藏`链接：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
<div class="dropdown-menu" aria-labelledby="navbarDropdown">
  .
  .
  .
  <a href="{{ route('products.favorites') }}" class="dropdown-item">我的收藏</a>
  .
  .
  ,
</div>
.
.
.
```

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/B0LqKX4d5i.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/B0LqKX4d5i.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 收藏商品列表"
```



