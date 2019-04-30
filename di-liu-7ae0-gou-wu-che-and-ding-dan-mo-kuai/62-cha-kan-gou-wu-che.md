## 查看购物车

上一节我们实现了将商品添加到购物车，本章节将要实现查看购物车中的商品。

## 1. 控制器

在`CartController`中添加`index()`方法：

_app/Http/Controllers/CartController.php_

```
    .
    .
    .
    public function index(Request $request)
    {
        $cartItems = $request->user()->cartItems()->with(['productSku.product'])->get();

        return view('cart.index', ['cartItems' => $cartItems]);
    }
.
.
.
```

`with(['productSku.product'])`方法用来预加载购物车里的商品和 SKU 信息。如果这里没有进行预加载而是在渲染模板时通过`$item->productSku->product`这种懒加载的方式，就会出现购物车中的每一项都要执行多次商品信息的 SQL 查询，导致单个页面执行的 SQL 数量过多，加载性能差的问题，也就是经典的 N + 1 查询问题。使用了预加载之后，Laravel 会通过类似`select * from product_skus where id in (xxxx)`的方式把原本需要多条 SQL 查询的数据用一条 SQL 就查到了，大大提升了执行效率。同时 Laravel 还支持通过`.`的方式加载多层级的关联关系，这里我们就通过`.`提前加载了与商品 SKU 关联的商品。

## 2. 路由

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::get('cart', 'CartController@index')->name('cart.index');
});
```

## 3. 前端模板

接下来创建一个新的前端模板：

```
$ mkdir -p resources/views/cart && touch resources/views/cart/index.blade.php
```

_resources/views/cart/index.blade.php_

```
@extends('layouts.app')
@section('title', '购物车')

@section('content')
<div class="row">
<div class="col-lg-10 offset-lg-1">
<div class="card">
  <div class="card-header">我的购物车</div>
  <div class="card-body">
    <table class="table table-striped">
      <thead>
      <tr>
        <th><input type="checkbox" id="select-all"></th>
        <th>商品信息</th>
        <th>单价</th>
        <th>数量</th>
        <th>操作</th>
      </tr>
      </thead>
      <tbody class="product_list">
      @foreach($cartItems as $item)
        <tr data-id="{{ $item->productSku->id }}">
          <td>
            <input type="checkbox" name="select" value="{{ $item->productSku->id }}" {{ $item->productSku->product->on_sale ? 'checked' : 'disabled' }}>
          </td>
          <td class="product_info">
            <div class="preview">
              <a target="_blank" href="{{ route('products.show', [$item->productSku->product_id]) }}">
                <img src="{{ $item->productSku->product->image_url }}">
              </a>
            </div>
            <div @if(!$item->productSku->product->on_sale) class="not_on_sale" @endif>
              <span class="product_title">
                <a target="_blank" href="{{ route('products.show', [$item->productSku->product_id]) }}">{{ $item->productSku->product->title }}</a>
              </span>
              <span class="sku_title">{{ $item->productSku->title }}</span>
              @if(!$item->productSku->product->on_sale)
                <span class="warning">该商品已下架</span>
              @endif
            </div>
          </td>
          <td><span class="price">￥{{ $item->productSku->price }}</span></td>
          <td>
            <input type="text" class="form-control form-control-sm amount" @if(!$item->productSku->product->on_sale) disabled @endif name="amount" value="{{ $item->amount }}">
          </td>
          <td>
            <button class="btn btn-sm btn-danger btn-remove">移除</button>
          </td>
        </tr>
      @endforeach
      </tbody>
    </table>
  </div>
</div>
</div>
</div>
@endsection
```

在循环输出购物车中的商品 SKU 时，我们把 SKU 的 ID 作为`<tr>`标签的`data-id`属性的值，这是为了之后删除商品或者提交订单时能够取得对应的 ID。

现实中很有可能出现用户先把商品添加到购物车，然后商家由于各种原因把商品下架了，因此我们还判断了对应的商品是否是上架状态，如果没有上架则禁止用户勾选对应的商品 SKU 并给出提示。

接下来需要给购物车配上样式：

_resources/sass/app.scss_

```
.
.
.
.cart-index-page {
  .product_list {
    .preview {
      img {
        max-width: 80px;
        max-height: 80px;
      }
      margin-right: 20px;
    }

    .product_info {
      display: flex;
      flex-direction: row;
      align-items: flex-start;
      font-size: 12px;
      .product_title, .sku_title, .warning {
        display: block;
      }

      .product_title a {
        color: #3c3c3c;
        font-size: 14px;
      }
      .not_on_sale {
        .product_title, .sku_title {
          text-decoration: line-through;
        }
        .warning {
          color: red;
        }
      }
    }

    .price {
      font-size: 12px;
      font-weight: 700;
      color: #3c3c3c;
    }

    input.amount {
      width: 60px;
      border-radius: 0 !important;
    }
  }
}
```

## 4. 测试

访问[http://shop.test/products](http://shop.test/products)，任意选择两个商品将其加入购物车，然后访问管理后台[http://shop.test/admin/products](http://shop.test/admin/products)，找到其中一个商品将其标记为下架状态，再访问购物车页面[http://shop.test/cart](http://shop.test/cart)：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Jt4IYxmeer.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Jt4IYxmeer.png!large)

可以看到被下架的商品无法勾选和修改数量，同时也给出了下架的提示。

## 从购物车中移除商品

接下来我们要实现`移除`按钮的功能。

## 1. 控制器

在`CartController`中添加`remove()`方法：

_app/Http/Controllers/CartController.php_

```
use App\Models\ProductSku;
.
.
.
    public function remove(ProductSku $sku, Request $request)
    {
        $request->user()->cartItems()->where('product_sku_id', $sku->id)->delete();

        return [];
    }
```

## 2. 路由

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::delete('cart/{sku}', 'CartController@remove')->name('cart.remove');
});
```

## 3. 前端页面

接下来我们需要在前端实现`移除`按钮的点击事件：

_resources/views/cart/index.blade.php_

```
.
.
.
@section('scriptsAfterJs')
<script>
  $(document).ready(function () {
    // 监听 移除 按钮的点击事件
    $('.btn-remove').click(function () {
      // $(this) 可以获取到当前点击的 移除 按钮的 jQuery 对象
      // closest() 方法可以获取到匹配选择器的第一个祖先元素，在这里就是当前点击的 移除 按钮之上的 <tr> 标签
      // data('id') 方法可以获取到我们之前设置的 data-id 属性的值，也就是对应的 SKU id
      var id = $(this).closest('tr').data('id');
      swal({
        title: "确认要将该商品移除？",
        icon: "warning",
        buttons: ['取消', '确定'],
        dangerMode: true,
      })
      .then(function(willDelete) {
        // 用户点击 确定 按钮，willDelete 的值就会是 true，否则为 false
        if (!willDelete) {
          return;
        }
        axios.delete('/cart/' + id)
          .then(function () {
            location.reload();
          })
      });
    });

  });
</script>
@endsection
```

现在刷新下页面试试效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/w9GQFvJYF1.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/w9GQFvJYF1.gif!large)

接下来我们需要实现全选和取消全选的功能：

_resources/views/cart/index.blade.php_

```
.
.
.
@section('scriptsAfterJs')
<script>
  $(document).ready(function () {
    .
    .
    .
    // 监听 全选/取消全选 单选框的变更事件
    $('#select-all').change(function() {
      // 获取单选框的选中状态
      // prop() 方法可以知道标签中是否包含某个属性，当单选框被勾选时，对应的标签就会新增一个 checked 的属性
      var checked = $(this).prop('checked');
      // 获取所有 name=select 并且不带有 disabled 属性的勾选框
      // 对于已经下架的商品我们不希望对应的勾选框会被选中，因此我们需要加上 :not([disabled]) 这个条件
      $('input[name=select][type=checkbox]:not([disabled])').each(function() {
        // 将其勾选状态设为与目标单选框一致
        $(this).prop('checked', checked);
      });
    });

  });
</script>
@endsection
```



## 4. 测试

为了更好测试所有的代码，因此要确保购物车里有被下架的商品：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/2WTk1aS0Tz.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/2WTk1aS0Tz.gif!large)

## 顶部添加购物车入口

接下来我们要在页面顶部增加一个购物车的图标，点击之后可以进入购物车页面。

由于 Boostrap4 默认没有引入 icon 相关的 css，因此我们需要自己引入，这里我们使用的是 FontAwesome：

```
$ yarn add @fortawesome/fontawesome-free
```

安装完成之后需要在样式文件最顶部中引入：

_resources/sass/app.scss_

```
@import "~@fortawesome/fontawesome-free/css/all.css";
.
.
.
```

然后我们就可以在模板里使用 FontAwesome 图标了：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
@guest
  <li class="nav-item"><a class="nav-link" href="{{ route('login') }}">登录</a></li>
  <li class="nav-item"><a class="nav-link" href="{{ route('register') }}">注册</a></li>
@else
  <li class="nav-item">
    <a class="nav-link mt-1" href="{{ route('cart.index') }}"><i class="fa fa-shopping-cart"></i></a>
  </li>
  .
  .
  .
@endguest
.
.
.
```

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/yavJGV65al.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/yavJGV65al.png!large)

## 加入购物车后跳转到购物车页面

我们之前点击`加入购物车`之后只弹了框告知操作成功，这样体验很不好，需要跳转到购物车页面：

_resources/views/products/show.blade.php_

```
.
.
.
$('.btn-add-to-cart').click(function () {
.
.
.
    .then(function () {
      swal('加入购物车成功', '', 'success')
        .then(function() {
          location.href = '{{ route('cart.index') }}';
        });
    },
    .
    .
    .
});
.
.
.
```

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/CnWzMJ911E.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/CnWzMJ911E.gif!large)

## Git 代码版本控制

由于我们引入了的 FontAwesome，webpack 在编译前端时会把 FontAwesome 的图标文件放到 public 目录下：

```
$ git status
```

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/mZBcomUdbF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/mZBcomUdbF.png!large)

所以我们需要将其排除：

_.gitignore_

```
.
.
.
/public/fonts
```

接着让我们将这些代码变更加入到版本控制中：

```
$ git add -A
$ git commit -m "购物车页面"
```



