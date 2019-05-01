## 评价商品

本章节我们将要实现评价已购商品的功能。

## 1. 控制器

由于我们需要对用户提交的数据进行校验，因此需要先创建一个`Request`类：

```
$ php artisan make:request SendReviewRequest
```

_app/Http/Requests/SendReviewRequest.php_

```
<?php

namespace App\Http\Requests;

use Illuminate\Validation\Rule;

class SendReviewRequest extends Request
{
    public function rules()
    {
        return [
            'reviews'          => ['required', 'array'],
            'reviews.*.id'     => [
                'required',
                Rule::exists('order_items', 'id')->where('order_id', $this->route('order')->id)
            ],
            'reviews.*.rating' => ['required', 'integer', 'between:1,5'],
            'reviews.*.review' => ['required'],
        ];
    }

    public function attributes()
    {
        return [
            'reviews.*.rating' => '评分',
            'reviews.*.review' => '评价',
        ];
    }
}
```

其中`$this->route('order')`可以获得当前路由对应的订单对象，`Rule::exists()`判断用户提交的 ID 是否属于此订单。

接下来在`OrdersController`里添加`review()`和`sendReview()`方法，分别代表展示评价页面和提交评价接口：

_app/Http/Controllers/OrdersController.php_

```
use Carbon\Carbon;
use App\Http\Requests\SendReviewRequest;
.
.
.
    public function review(Order $order)
    {
        // 校验权限
        $this->authorize('own', $order);
        // 判断是否已经支付
        if (!$order->paid_at) {
            throw new InvalidRequestException('该订单未支付，不可评价');
        }
        // 使用 load 方法加载关联数据，避免 N + 1 性能问题
        return view('orders.review', ['order' => $order->load(['items.productSku', 'items.product'])]);
    }

    public function sendReview(Order $order, SendReviewRequest $request)
    {
        // 校验权限
        $this->authorize('own', $order);
        if (!$order->paid_at) {
            throw new InvalidRequestException('该订单未支付，不可评价');
        }
        // 判断是否已经评价
        if ($order->reviewed) {
            throw new InvalidRequestException('该订单已评价，不可重复提交');
        }
        $reviews = $request->input('reviews');
        // 开启事务
        \DB::transaction(function () use ($reviews, $order) {
            // 遍历用户提交的数据
            foreach ($reviews as $review) {
                $orderItem = $order->items()->find($review['id']);
                // 保存评分和评价
                $orderItem->update([
                    'rating'      => $review['rating'],
                    'review'      => $review['review'],
                    'reviewed_at' => Carbon::now(),
                ]);
            }
            // 将订单标记为已评价
            $order->update(['reviewed' => true]);
        });    

        return redirect()->back();
    }
```

## 2. 路由

然后添加对应的路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
   Route::get('orders/{order}/review', 'OrdersController@review')->name('orders.review.show');
   Route::post('orders/{order}/review', 'OrdersController@sendReview')->name('orders.review.store');
});
```

## 3. 前端模板

接下来我们创建一个新的前端模板文件用户展示评价页面：

```
$ touch resources/views/orders/review.blade.php
```

_resources/views/orders/review.blade.php_

```
@extends('layouts.app')
@section('title', '商品评价')

@section('content')
<div class="row">
<div class="col-lg-10 offset-lg-1">
<div class="card">
  <div class="card-header">
    商品评价
    <a class="float-right" href="{{ route('orders.index') }}">返回订单列表</a>
  </div>
  <div class="card-body">
    <form action="{{ route('orders.review.store', [$order->id]) }}" method="post">
      <input type="hidden" name="_token" value="{{ csrf_token() }}">
      <table class="table">
        <tbody>
        <tr>
          <td>商品名称</td>
          <td>打分</td>
          <td>评价</td>
        </tr>
        @foreach($order->items as $index => $item)
          <tr>
            <td class="product-info">
              <div class="preview">
                <a target="_blank" href="{{ route('products.show', [$item->product_id]) }}">
                  <img src="{{ $item->product->image_url }}">
                </a>
              </div>
              <div>
                <span class="product-title">
                   <a target="_blank" href="{{ route('products.show', [$item->product_id]) }}">{{ $item->product->title }}</a>
                </span>
                <span class="sku-title">{{ $item->productSku->title }}</span>
              </div>
              <input type="hidden" name="reviews[{{$index}}][id]" value="{{ $item->id }}">
            </td>
            <td class="vertical-middle">
              <!-- 如果订单已经评价则展示评分，下同 -->
              @if($order->reviewed)
                <span class="rating-star-yes">{{ str_repeat('★', $item->rating) }}</span><span class="rating-star-no">{{ str_repeat('★', 5 - $item->rating) }}</span>
              @else
                <ul class="rate-area">
                  <input type="radio" id="5-star-{{$index}}" name="reviews[{{$index}}][rating]" value="5" checked /><label for="5-star-{{$index}}"></label>
                  <input type="radio" id="4-star-{{$index}}" name="reviews[{{$index}}][rating]" value="4" /><label for="4-star-{{$index}}"></label>
                  <input type="radio" id="3-star-{{$index}}" name="reviews[{{$index}}][rating]" value="3" /><label for="3-star-{{$index}}"></label>
                  <input type="radio" id="2-star-{{$index}}" name="reviews[{{$index}}][rating]" value="2" /><label for="2-star-{{$index}}"></label>
                  <input type="radio" id="1-star-{{$index}}" name="reviews[{{$index}}][rating]" value="1" /><label for="1-star-{{$index}}"></label>
                </ul>
              @endif
            </td>
            <td>
              @if($order->reviewed)
                {{ $item->review }}
              @else
                <textarea class="form-control {{ $errors->has('reviews.'.$index.'.review') ? 'is-invalid' : '' }}" name="reviews[{{$index}}][review]"></textarea>
                @if($errors->has('reviews.'.$index.'.review'))
                  @foreach($errors->get('reviews.'.$index.'.review') as $msg)
                    <span class="invalid-feedback" role="alert"><strong>{{ $msg }}</strong></span>
                  @endforeach
                @endif
              @endif
            </td>
          </tr>
        @endforeach
        </tbody>
        <tfoot>
        <tr>
          <td colspan="3" class="text-center">
            @if(!$order->reviewed)
              <button type="submit" class="btn btn-primary center-block">提交</button>
            @else
              <a href="{{ route('orders.show', [$order->id]) }}" class="btn btn-primary">查看订单</a>
            @endif
          </td>
        </tr>
        </tfoot>
      </table>
    </form>
  </div>
</div>
</div>
</div>
@endsection
```

然后是样式文件：

_resources/sass/app.scss_

```
.
.
.
.orders-review-show-page {
  .vertical-middle {
    vertical-align: middle;
  }
  .product-info {
    display: flex;
    flex-direction: row;
    .preview {
      img {
        max-width: 60px;
        max-height: 60px;
      }
      border: 1px solid #eee;
      margin-right: 5px;
    }
    .product-title, .sku-title {
      display: block;
    }
    .product-title > a {
      color: #3c3c3c;
    }
    .sku-title {
      color: #9e9e9e;
    }
  }
  .rating-star-yes {
    color: gold;
    font-size: 150%;
  }
  .rating-star-no {
    color: lightgrey;
    font-size: 150%;
  }
  .rate-area {
    float: left;
    border-style: none;
    margin: 0 auto;
    padding: 0;
    &:not(:checked) {
      & > input {
        position: absolute;
        top: -9999px;
        clip: rect(0,0,0,0);
      }
      & > label {
        float: right;
        width: 1em;
        overflow: hidden;
        white-space: nowrap;
        cursor: pointer;
        line-height: 1.2;
        color: lightgrey;
        font-size: 150%;

        &:before {
          content: '★ ';
        }
      }
      & > label:hover, & > label:hover ~ label {
        color: gold;
      }
    }
    & > input:checked ~ label {
      color: gold;
    }
    & > label:active {
      position: relative;
      top: 2px;
      left: 2px;
    }

    & > input:checked + label:hover, & > input:checked + label:hover ~ label, & > input:checked ~ label:hover, & > input:checked ~ label:hover ~ label, & > label:hover ~ input:checked ~ label {
      color: gold;
    }
  }

  textarea {
    border-radius: 0px;
  }
}
```

## 4. 测试

接下来我们测试一下，访问[http://shop.test/orders/{订单 ID}/review](http://shop.test/orders/%7B%E8%AE%A2%E5%8D%95ID%7D/review)：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/t5oXUUOgjM.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/t5oXUUOgjM.png!large)

试一下直接提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/fRlzLGDIXX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/fRlzLGDIXX.png!large)

填入一些字符之后再次提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lNb375uuX4.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lNb375uuX4.png!large)

  


