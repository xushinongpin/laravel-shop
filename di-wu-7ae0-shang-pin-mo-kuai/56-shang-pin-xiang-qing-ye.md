[![](https://iocaffcdn.phphub.org/uploads/images/201806/03/5320/BMeCGp66gg.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/03/5320/BMeCGp66gg.png?imageView2/2/w/1240/h/0)

## 商品详情页

本章节将要实现商品详情页的展示（如上图）。

## 1. 控制器

在`ProductsController`类中新增`show()`方法：

_app/Http/Controllers/ProductsController.php_

```
.
.
.
    public function show(Product $product, Request $request)
    {
        // 判断商品是否已经上架，如果没有上架则抛出异常。
        if (!$product->on_sale) {
            throw new \Exception('商品未上架');
        }

        return view('products.show', ['product' => $product]);
    }
```

## 2. 模板页面

接下来我们创建一个模板页面：

```
$ touch resources/views/products/show.blade.php
```

_resources/views/products/show.blade.php_

```
@extends('layouts.app')
@section('title', $product->title)

@section('content')
<div class="row">
<div class="col-lg-10 offset-lg-1">
<div class="card">
  <div class="card-body product-info">
    <div class="row">
      <div class="col-5">
        <img class="cover" src="{{ $product->image_url }}" alt="">
      </div>
      <div class="col-7">
        <div class="title">{{ $product->title }}</div>
        <div class="price"><label>价格</label><em>￥</em><span>{{ $product->price }}</span></div>
        <div class="sales_and_reviews">
          <div class="sold_count">累计销量 <span class="count">{{ $product->sold_count }}</span></div>
          <div class="review_count">累计评价 <span class="count">{{ $product->review_count }}</span></div>
          <div class="rating" title="评分 {{ $product->rating }}">评分 <span class="count">{{ str_repeat('★', floor($product->rating)) }}{{ str_repeat('☆', 5 - floor($product->rating)) }}</span></div>
        </div>
        <div class="skus">
          <label>选择</label>
          <div class="btn-group btn-group-toggle" data-toggle="buttons">
            @foreach($product->skus as $sku)
              <label class="btn sku-btn" title="{{ $sku->description }}" >
                <input type="radio" name="skus" autocomplete="off" value="{{ $sku->id }}"> {{ $sku->title }}
              </label>
            @endforeach
          </div>
        </div>
        <div class="cart_amount"><label>数量</label><input type="text" class="form-control form-control-sm" value="1"><span>件</span><span class="stock"></span></div>
        <div class="buttons">
          <button class="btn btn-success btn-favor">❤ 收藏</button>
          <button class="btn btn-primary btn-add-to-cart">加入购物车</button>
        </div>
      </div>
    </div>
    <div class="product-detail">
      <ul class="nav nav-tabs" role="tablist">
        <li class="nav-item">
          <a class="nav-link active" href="#product-detail-tab" aria-controls="product-detail-tab" role="tab" data-toggle="tab" aria-selected="true">商品详情</a>
        </li>
        <li class="nav-item">
          <a class="nav-link" href="#product-reviews-tab" aria-controls="product-reviews-tab" role="tab" data-toggle="tab" aria-selected="false">用户评价</a>
        </li>
      </ul>
      <div class="tab-content">
        <div role="tabpanel" class="tab-pane active" id="product-detail-tab">
          {!! $product->description !!}
        </div>
        <div role="tabpanel" class="tab-pane" id="product-reviews-tab">
        </div>
      </div>
    </div>
  </div>
</div>
</div>
</div>
@endsection
```

代码解析：

* `<`
  `div class="btn-group btn-group-toggle" data-toggle="buttons"`
  `>`
  ` ... `
  `<`
  `/div`
  `>`
  这里使用了 Bootstrap 的按钮组来输出 SKU 列表。
* `<`
  `ul class="nav nav-tabs" role="tablist"`
  `>`
  ` ... `
  `<`
  `/ul`
  `>`
  以及
  `<`
  `div class="tab-content"`
  `>`
  ` ... `
  `<`
  `/div`
  `>`
  是 Bootstrap 的 Tab 插件，我们用来输出商品详情和评价列表，由于我们尚未涉及评价相关的逻辑因此这里暂时留空。
* `{!! $product-`
  `>`
  `description !!}`
  因为我们后台编辑商品详情用的是富文本编辑器，提交的内容是 Html 代码，此处需要原样输出而不需要进行 Html 转义。

## 3. 样式美化

接下来添加一下相关的样式：

_resources/sass/app.scss_

```
.
.
.
.products-show-page {
   .cover {
     width: 100%;
     border: solid 1px #eee;
     padding: 30px 0;
   }
   .title {
     font-size: 24px;
     font-weight: bold;
     margin-bottom: 10px;
   }
   .price {
     label {
       width: 69px;
       color: #999;
       font-size: 12px;
       padding-left: 10px;
     }
     em {
       font-family: Arial;
       font-size: 18px;
       font-style: normal;
       text-decoration: none;
       vertical-align: middle;
     }
     span {
       font-family: Arial;
       font-size: 24px;
       font-weight: bolder;
       text-decoration: none;
       vertical-align: middle;
     }
     line-height: 30px;
     background-color: #e9e9e9;
     color: red;
     font-size: 20px;
   }
   .sales_and_reviews {
     border-top: 1px dotted #c9c9c9;
     border-bottom: 1px dotted #c9c9c9;
     margin: 5px 0 10px;
     display: flex;
     flex-direction: row;
     font-size: 12px;
     &>div {
       &.sold_count,&.review_count {
         border-right: 1px dotted #c9c9c9;
       }
       width: 33%;
       text-align: center;
       padding: 5px;
       .count {
         color: #FF0036;
         font-weight: 700;
       }
     }
   }
   .skus {
     &>label {
       color: #999;
       font-size: 12px;
       padding: 0 10px 0 10px;
     }
     .btn-group {
       margin-left: -10px;
       label {
         border-radius: 0 !important;
         margin: 1px 5px;
         padding: 2px 5px;
         font-size: 12px;
       }
       .btn {
         border: 1px solid #ccc;
       }
       .btn.active, .btn:hover {
         margin-top: 0px !important;
         background: #fff !important;
         border: 2px solid red !important;
       }
       .btn.focus {
         outline: 0 !important;
       }
     }
   }
   .cart_amount {
     label {
       color: #999;
       font-size: 12px;
       padding: 0 10px 0 10px;
     }
     font-size: 12px;
     color: #888;
     margin: 10px 0 20px;
     input {
       width: 50px;
       display: inline-block;
       border-radius: 0 !important;
     }
     span {
       color: #999;
       font-size: 12px;
       padding-left: 10px;
     }
   }
   .buttons {
     padding-left: 44px;
   }

   .product-detail  {
     .nav.nav-tabs > li > a {
       border-radius: 0 !important;
     }
     margin: 20px 0;
     .tab-content {
       border: 1px solid #eee;
       padding: 20px;
     }
   }
 }
```

4. 路由

接下来添加路由：

_routes/web.php_

```
Route::get('products', 'ProductsController@index')->name('products.index');
Route::get('products/{product}', 'ProductsController@show')->name('products.show');
.
.
.
```

> 我们希望访客也能访问，因此这个路由**不需要**放在有`auth`中间件的路由组中。

## 5. 测试

现在直接在浏览器中输入[http://shop.test/products/2](http://shop.test/products/2)来访问商品详情页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/x6NH2mtLiJ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/x6NH2mtLiJ.png!large)

## 6. 优化

但是现在商品价格是写死的，当用户点击各个 SKU 的时候价格和剩余库存并没有变化：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/nttBkhjQDz.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/nttBkhjQDz.gif!large)

我们需要用 JS 来实现这个功能：

_resources/views/products/show.blade.php_

```
.
.
.
@foreach($product->skus as $sku)
  <label
      class="btn sku-btn"
      data-price="{{ $sku->price }}"
      data-stock="{{ $sku->stock }}"
      data-toggle="tooltip"
      title="{{ $sku->description }}"
      data-placement="bottom">
    <input type="radio" name="skus" autocomplete="off" value="{{ $sku->id }}"> {{ $sku->title }}
  </label>
@endforeach
.
.
.
@section('scriptsAfterJs')
<script>
  $(document).ready(function () {
    $('[data-toggle="tooltip"]').tooltip({trigger: 'hover'});
    $('.sku-btn').click(function () {
      $('.product-info .price span').text($(this).data('price'));
      $('.product-info .stock').text('库存：' + $(this).data('stock') + '件');
    });
  });
</script>
@endsection
```

代码解析：

* 在输出 SKU 的按钮组的时候，我们通过
  `data-*`
  属性把对应 SKU 的价格和剩余库存放在了 Html 标签里。
* 同时加上了
  `data-toggle="tooltip"`
  这个属性来启用 Bootstrap 的工具提示来美化样式。
* 在 JS 代码中我们监听了
  `.sku-btn`
  的点击事件，当用户点击 SKU 时，我们从对应按钮的
  `data-*`
  属性取出价格和库存并输出到对应的 Html 标签中。

刷新页面看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/GMIEEz6Czw.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/GMIEEz6Czw.gif!large)

## 7. 商品列表页添加链接

接下来我们需要把商品详情页的链接添加到商品列表页：

_resources/views/products/index.blade.php_

```
.
.
.
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
              <div class="title">
                <a href="{{ route('products.show', ['product' => $product->id]) }}">{{ $product->title }}</a>
              </div>
            </div>
        .
        .
        .
```

> 我们给图片和标题都加上了链接，这样用户体验更好。

最后看一下效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/vXQlBj4VuJ.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/vXQlBj4VuJ.gif!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 商品详情页"

```



