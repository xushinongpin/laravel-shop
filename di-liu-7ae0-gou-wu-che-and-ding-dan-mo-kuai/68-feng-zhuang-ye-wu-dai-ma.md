[![](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/qWmFFH1eIc.jpeg?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/qWmFFH1eIc.jpeg?imageView2/2/w/1240/h/0)

## 封装业务代码

在本章节我们已经完成了购物车功能和下单功能，但是我们会发现我们在 Controller 里面写了大量的包含复杂逻辑的业务代码，这是一个坏习惯，这样子随着需求的增加，我们的控制器很快就变得臃肿。如果以后我们要开发 App 端，这些代码可能需要在 Api 的 Controller 里再重复一遍，假如出现业务逻辑的调整就需要修改两个或更多地方，这明显是不合理的。因此我们需要对**逻辑复杂**的**业务代码**进行封装。

这里我们将在项目里采用 Service 模式来封装代码。购物车的逻辑，放置于`CartService`类里，将下单的业务逻辑代码放置于`OrderService`里。

## 一、购物车

首先创建一个`CartService`类：

```
$ mkdir -p app/Services && touch app/Services/CartService.php
```

_app/Services/CartService.php_

```
<?php

namespace App\Services;

use Auth;
use App\Models\CartItem;

class CartService
{
    public function get()
    {
        return Auth::user()->cartItems()->with(['productSku.product'])->get();
    }

    public function add($skuId, $amount)
    {
        $user = Auth::user();
        // 从数据库中查询该商品是否已经在购物车中
        if ($item = $user->cartItems()->where('product_sku_id', $skuId)->first()) {
            // 如果存在则直接叠加商品数量
            $item->update([
                'amount' => $item->amount + $amount,
            ]);
        } else {
            // 否则创建一个新的购物车记录
            $item = new CartItem(['amount' => $amount]);
            $item->user()->associate($user);
            $item->productSku()->associate($skuId);
            $item->save();
        }

        return $item;
    }

    public function remove($skuIds)
    {
        // 可以传单个 ID，也可以传 ID 数组
        if (!is_array($skuIds)) {
            $skuIds = [$skuIds];
        }
        Auth::user()->cartItems()->whereIn('product_sku_id', $skuIds)->delete();
    }
}
```

可以看到我们在这里直接通过`Auth::user()`的方式获取用户，这是因为通常来说只有当前用户才会操作自己的购物车，所以可以不需要从外部传入`$user`对象。

`remove()`方法的参数可以传入单个 ID 也能传入一个 ID 数组，这个是 Laravel 中很常见的设计，比如 Model 对象里面的`with()`、`load()`等方法都是支持只传一个值和一个数组的，这样调用起来会十分方便。

接下来我们要修改`CartController`，将其改为调用刚刚创建的`CartService`类：

_app/Http/Controllers/CartController.php_

```
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Http\Requests\AddCartRequest;
use App\Models\ProductSku;
use App\Services\CartService;

class CartController extends Controller
{
    protected $cartService;

    // 利用 Laravel 的自动解析功能注入 CartService 类
    public function __construct(CartService $cartService)
    {
        $this->cartService = $cartService;
    }

    public function index(Request $request)
    {
        $cartItems = $this->cartService->get();
        $addresses = $request->user()->addresses()->orderBy('last_used_at', 'desc')->get();

        return view('cart.index', ['cartItems' => $cartItems, 'addresses' => $addresses]);
    }

    public function add(AddCartRequest $request)
    {
        $this->cartService->add($request->input('sku_id'), $request->input('amount'));

        return [];
    }

    public function remove(ProductSku $sku, Request $request)
    {
        $this->cartService->remove($sku->id);

        return [];
    }
}
```

这里我们使用了 Laravel 容器的自动解析功能，当 Laravel 初始化 Controller 类时会检查该类的构造函数参数，在本例中 Laravel 会自动创建一个`CartService`对象作为构造参数传入给`CartController`。

我们在`OrdersController`也进行了购物车的删除动作，因此这里也需要调整一下：

_app/Http/Controllers/OrdersController.php_

```
use App\Services\CartService;
.
.
.
    // 利用 Laravel 的自动解析功能注入 CartService 类
    public function store(OrderRequest $request, CartService $cartService)
    {
        $user  = $request->user();
        // 别忘了把 $cartService 加入 use 中
        $order = \DB::transaction(function () use ($user, $request, $cartService) {
            .
            .
            .
            // 将下单的商品从购物车中移除
            $skuIds = collect($request->input('items'))->pluck('sku_id')->all();
            $cartService->remove($skuIds);
            .
            .
            .
```

可以看到与`CartController`通过构造函数参数注入的方式不同，这里我们选择了在方法的参数中注入`CartService`类的对象。这是因为在`CartController`中所有的方法都会用到`CartService`，把`CartService`作为类中的一个属性在调用起来会方便许多；而`OrdersController`的大多数方法与`CartService`无关，如果把`CartService`对象作为一个属性，那么在请求与`CartService`无关的接口时就会做一些无用的创建类操作，因此我们选择只在`store()`这个方法的参数里注入。

通过上面两个例子，我们可以看出来 Laravel 的依赖自动注入无处不在，在我们的开发过程中带来的极大的便利。

**做完修改之后一定不要忘了测试一遍有修改的地方。**下面是需要测试的功能点：

1. 将商品加入购物车；
2. 查看购物车页面；
3. 删除购物车中的商品；
4. 将购物车中的商品提交为订单，并确认提交成功之后购物车中对应的商品被移除。

## 二、订单

接下来我们要封装的是订单的相关功能，首先创建`OrderService`类：

```
$ touch app/Services/OrderService.php
```

_app/Services/OrderService.php_

```
<?php

namespace App\Services;

use App\Models\User;
use App\Models\UserAddress;
use App\Models\Order;
use App\Models\ProductSku;
use App\Exceptions\InvalidRequestException;
use App\Jobs\CloseOrder;
use Carbon\Carbon;

class OrderService
{
    public function store(User $user, UserAddress $address, $remark, $items)
    {
        // 开启一个数据库事务
        $order = \DB::transaction(function () use ($user, $address, $remark, $items) {
            // 更新此地址的最后使用时间
            $address->update(['last_used_at' => Carbon::now()]);
            // 创建一个订单
            $order   = new Order([
                'address'      => [ // 将地址信息放入订单中
                    'address'       => $address->full_address,
                    'zip'           => $address->zip,
                    'contact_name'  => $address->contact_name,
                    'contact_phone' => $address->contact_phone,
                ],
                'remark'       => $remark,
                'total_amount' => 0,
            ]);
            // 订单关联到当前用户
            $order->user()->associate($user);
            // 写入数据库
            $order->save();

            $totalAmount = 0;
            // 遍历用户提交的 SKU
            foreach ($items as $data) {
                $sku  = ProductSku::find($data['sku_id']);
                // 创建一个 OrderItem 并直接与当前订单关联
                $item = $order->items()->make([
                    'amount' => $data['amount'],
                    'price'  => $sku->price,
                ]);
                $item->product()->associate($sku->product_id);
                $item->productSku()->associate($sku);
                $item->save();
                $totalAmount += $sku->price * $data['amount'];
                if ($sku->decreaseStock($data['amount']) <= 0) {
                    throw new InvalidRequestException('该商品库存不足');
                }
            }
            // 更新订单总金额
            $order->update(['total_amount' => $totalAmount]);

            // 将下单的商品从购物车中移除
            $skuIds = collect($items)->pluck('sku_id')->all();
            app(CartService::class)->remove($skuIds);

            return $order;
        });

        // 这里我们直接使用 dispatch 函数
        dispatch(new CloseOrder($order, config('app.order_ttl')));

        return $order;
    }
}
```

这里大多数的代码都是从`OrdersController`中直接复制过来的，只有些许的变化需要注意：

1. `$user`
   、
   `$address`
   变量改为从参数获取。我们在封装功能的时候有一点一定要注意，
   `$request`
   不可以出现在控制器和中间件以外的地方，根据职责单一原则，获取数据这个任务应该由控制器来完成，封装的类只需要专注于业务逻辑的实现。
2. `CartService`
   的调用方式改为了通过
   `app()`
   函数创建，因为这个
   `store()`
   方法是我们手动调用的，无法通过 Laravel 容器的自动解析来注入。在我们代码里调用封装的库时一定
   **不可以**
   使用
   `new`
   关键字来初始化，而是应该通过 Laravel 的容器来初始化，因为在之后的开发过程中
   `CartService`
   类的构造函数可能会发生变化，比如注入了其他的类，如果我们使用
   `new`
   来初始化的话，就需要在每个调用此类的地方进行修改；而使用
   `app()`
   或者自动解析注入等方式 Laravel 则会自动帮我们处理掉这些依赖。
3. 之前在控制器中可以通过
   `$this-`
   `>`
   `dispatch()`
   方法来触发任务类，但在我们的封装的类中并没有这个方法，因此关闭订单的任务类改为
   `dispatch()`
   辅助函数来触发。

接下来修改控制器：

_app/Http/Controllers/OrdersController.php_

```
<?php

namespace App\Http\Controllers;

use App\Http\Requests\OrderRequest;
use App\Models\UserAddress;
use App\Models\Order;
use Illuminate\Http\Request;
use App\Services\OrderService;

class OrdersController extends Controller
{
    .
    .
    .
    public function store(OrderRequest $request, OrderService $orderService)
    {
        $user    = $request->user();
        $address = UserAddress::find($request->input('address_id'));

        return $orderService->store($user, $address, $request->input('remark'), $request->input('items'));
    }
}
```

与之前`OrderService`同理，这里通过 Laravel 的自动解析来注入`OrderService`。

最后别忘了测试一下我们的代码，这里就只需要测试一下在购物车页面创建订单是否正常即可。

## 关于 Service 模式

Service 模式将 PHP 的商业逻辑写在对应责任的 Service 类里，解決 Controller 臃肿的问题。并且符合 SOLID 的单一责任原则，购物车的逻辑由`CartService`负责，而不是`CartController`，控制器是调度中心，编码逻辑更加清晰。后面如果我们有 API 或者其他会使用到购物车功能的需求，也可以直接使用`CartService`，代码可复用性大大增加。再加上 Service 可以利用 Laravel 提供的依赖注入机制，大大提高了`Service`部分代码的可测试性，程序的健壮性越佳。

> 推荐一篇 Service 模式的文章 ——[如何使用 Service 模式？](https://www.kancloud.cn/curder/laravel/408485)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "封装业务代码"
```



