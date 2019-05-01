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

接下来我们测试一下，访问[http://shop.test/orders/{订单 ID}/review](http://shop.test/orders/{订单ID}/review)：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/t5oXUUOgjM.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/t5oXUUOgjM.png!large)

试一下直接提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/fRlzLGDIXX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/fRlzLGDIXX.png!large)

填入一些字符之后再次提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lNb375uuX4.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/lNb375uuX4.png!large)

## 5. 更新商品评分

用户给商品打完分之后，系统需要重新计算对应商品的评分数据，这里我们还是通过事件系统来实现。

首先创建一个订单已评价`OrderReviewed`事件：

```
$ php artisan make:event OrderReviewed
```

和订单已支付`OrderPaid`事件一样，只需要包含订单数据即可：

_app/Events/OrderReviewed.php_

```
use App\Models\Order;
.
.
.
class OrderReviewed
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

接下来创建对应的事件监听器`UpdateProductRating`：

```
$ php artisan make:listener UpdateProductRating --event=OrderReviewed
```

_app/Listeners/UpdateProductRating.php_

```
<?php

namespace App\Listeners;

use DB;
use App\Models\OrderItem;
use App\Events\OrderReviewed;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;

// implements ShouldQueue 代表这个事件处理器是异步的
class UpdateProductRating implements ShouldQueue
{
    public function handle(OrderReviewed $event)
    {
        // 通过 with 方法提前加载数据，避免 N + 1 性能问题
        $items = $event->getOrder()->items()->with(['product'])->get();
        foreach ($items as $item) {
            $result = OrderItem::query()
                ->where('product_id', $item->product_id)
                ->whereHas('order', function ($query) {
                    $query->whereNotNull('paid_at');
                })
                ->first([
                    DB::raw('count(*) as review_count'),
                    DB::raw('avg(rating) as rating')
                ]);
            // 更新商品的评分和评价数
            $item->product->update([
                'rating'       => $result->rating,
                'review_count' => $result->review_count,
            ]);
        }
    }
}
```

重点看一下`first()`方法，`first()`方法接受一个数组作为参数，代表此次 SQL 要查询出来的字段，默认情况下 Laravel 会给数组里面的值的两边加上`````````这个符号，比如`````first\(\['name', 'email'\]\)\`生成的 SQL 会类似：

    select `name`, `email` from xxx

所以如果我们直接传入`first(['count(*) as review_count', 'avg(rating) as rating'])`，最后生成的 SQL 肯定是不正确的。这里我们用`DB::raw()`方法来解决这个问题，Laravel 在构建 SQL 的时候如果遇到`DB::raw()`就会把`DB::raw()`的参数原样拼接到 SQL 里。

接下来在`EventServiceProvider`注册事件和处理的关联：

_app/Providers/EventServiceProvider.php_

```
use App\Events\OrderReviewed;
use App\Listeners\UpdateProductRating;
.
.
.
    protected $listen = [
        .
        .
        .
        OrderReviewed::class => [
            UpdateProductRating::class,
        ],
    ];
```

最后我们在控制器里触发这个事件：

_app/Http/Controllers/OrdersController.php_

```
use App\Events\OrderReviewed;
.
.
.
    public function sendReview(Order $order, SendReviewRequest $request)
    {
        .
        .
        .
        \DB::transaction(function () use ($reviews, $order) {
            .
            .
            .
            event(new OrderReviewed($order));
        });
        .
        .
        .
    }
```

接下来我们测试一下这个事件和对应的处理器。

> 别忘记启动队列处理器`php artisan queue:work`

首先我们用相同的商品创建一笔订单，然后在数据库中手动修改这笔订单的`paid_at`字段使其成为已支付订单，再进入该订单的评价页面提交一个新的评价，看看队列处理器：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/KFqCRZTWD3.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/KFqCRZTWD3.png!large)

然后到数据库里看看`order_items`表：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Kp9vgfU0Nz.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Kp9vgfU0Nz.png!large)

可以看到`product_id=3`的商品有两个评价，分别是 3 分和 5 分，接下来去`products`表看看对应商品的记录：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/QEn9ftfyKv.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/QEn9ftfyKv.png!large)

评分 4，评价数 2，符合我们的情况。



## 6. 添加入口

接下来我们要把评价的入口放在订单列表页面：

_resources/views/orders/index.blade.php_

```
.
.
.
<td rowspan="{{ count($order->items) }}" class="text-center">
  <a class="btn btn-primary btn-sm" href="{{ route('orders.show', ['order' => $order->id]) }}">查看订单</a>
  <!-- 评价入口开始 -->
  @if($order->paid_at)
  <a class="btn btn-success btn-sm" href="{{ route('orders.review.show', ['order' => $order->id]) }}">
  {{ $order->reviewed ? '查看评价' : '评价' }}
  </a>
  @endif
  <!-- 评价入口结束 -->
</td>
.
.
.
```

访问订单列表查看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/FKyPQlQAqD.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/FKyPQlQAqD.png!large)

## 7. 在商品页展示

我们在写商品展示页面的时候预留了商品评价的列表，现在来填充这个地方。

首先在控制器里加载评价：

_app/Http/Controllers/ProductsController.php_

```
use App\Models\OrderItem;
.
.
.
    public function show(Product $product, Request $request)
    {
        .
        .
        .
        $reviews = OrderItem::query()
            ->with(['order.user', 'productSku']) // 预先加载关联关系
            ->where('product_id', $product->id)
            ->whereNotNull('reviewed_at') // 筛选出已评价的
            ->orderBy('reviewed_at', 'desc') // 按评价时间倒序
            ->limit(10) // 取出 10 条
            ->get();

        // 最后别忘了注入到模板中
        return view('products.show', [
            'product' => $product,
            'favored' => $favored,
            'reviews' => $reviews
        ]);
    }
```

然后在商品详情页的模板中循环展示：

_resources/views/products/show.blade.php_

```
.
.
.
<div role="tabpanel" class="tab-pane" id="product-reviews-tab">
  <!-- 评论列表开始 -->
  <table class="table table-bordered table-striped">
    <thead>
    <tr>
      <td>用户</td>
      <td>商品</td>
      <td>评分</td>
      <td>评价</td>
      <td>时间</td>
    </tr>
    </thead>
    <tbody>
      @foreach($reviews as $review)
      <tr>
        <td>{{ $review->order->user->name }}</td>
        <td>{{ $review->productSku->title }}</td>
        <td>{{ str_repeat('★', $review->rating) }}{{ str_repeat('☆', 5 - $review->rating) }}</td>
        <td>{{ $review->review }}</td>
        <td>{{ $review->reviewed_at->format('Y-m-d H:i') }}</td>
      </tr>
      @endforeach
    </tbody>
  </table>
  <!-- 评论列表结束 -->
</div>
.
.
.
```

现在进到商品详情页看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/M1Te94YvVp.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/M1Te94YvVp.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "评价商品"
```



