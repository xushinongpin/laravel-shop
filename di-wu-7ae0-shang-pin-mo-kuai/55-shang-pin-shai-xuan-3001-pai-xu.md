## 商品筛选、排序

电商网站必不可缺的功能就是商品的筛选和排序，接下来我们就要实现这个功能。

首先我们在前端页面加上筛选和排序的组件：

## 1. 前端组件

_resources/views/products/index.blade.php_

```
.
.
.
<div class="card">
  <div class="card-body">
    <!-- 筛选组件开始 -->
    <form action="{{ route('products.index') }}" class="search-form">
      <div class="form-row">
        <div class="col-md-9">
          <div class="form-row">
            <div class="col-auto"><input type="text" class="form-control form-control-sm" name="search" placeholder="搜索"></div>
            <div class="col-auto"><button class="btn btn-primary btn-sm">搜索</button></div>
          </div>
        </div>
        <div class="col-md-3">
          <select name="order" class="form-control form-control-sm float-right">
            <option value="">排序方式</option>
            <option value="price_asc">价格从低到高</option>
            <option value="price_desc">价格从高到低</option>
            <option value="sold_count_desc">销量从高到低</option>
            <option value="sold_count_asc">销量从低到高</option>
            <option value="rating_desc">评价从高到低</option>
            <option value="rating_asc">评价从低到高</option>
          </select>
        </div>
      </div>
    </form>
    <!-- 筛选组件结束 -->
    <div class="row products-list">
    .
    .
    .
  </div>
</div>
```

刷新页面看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/Iyuy6383Kl.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/Iyuy6383Kl.png!large)

发现搜索组件样式不好看，修改下样式文件，在两边和下方加上空隙：

_resources/sass/app.scss_

```
.products-index-page {
    .
    .
    .
  .search-form {
    padding: 0 5px 10px;
    margin-bottom: 10px;
    border-bottom: 1px solid #eee;
  }
}
```

再看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/9VAEXw8NRx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/9VAEXw8NRx.png!large)

## 2. 修改控制器

接下来需要让我们的控制器支持商品的筛选和排序：

_app/Http/Controllers/ProductsController.php_

```
.
.
.
    public function index(Request $request)
    {
        // 创建一个查询构造器
        $builder = Product::query()->where('on_sale', true);
        // 判断是否有提交 search 参数，如果有就赋值给 $search 变量
        // search 参数用来模糊搜索商品
        if ($search = $request->input('search', '')) {
            $like = '%'.$search.'%';
            // 模糊搜索商品标题、商品详情、SKU 标题、SKU描述
            $builder->where(function ($query) use ($like) {
                $query->where('title', 'like', $like)
                    ->orWhere('description', 'like', $like)
                    ->orWhereHas('skus', function ($query) use ($like) {
                        $query->where('title', 'like', $like)
                            ->orWhere('description', 'like', $like);
                    });
            });
        }

        // 是否有提交 order 参数，如果有就赋值给 $order 变量
        // order 参数用来控制商品的排序规则
        if ($order = $request->input('order', '')) {
            // 是否是以 _asc 或者 _desc 结尾
            if (preg_match('/^(.+)_(asc|desc)$/', $order, $m)) {
                // 如果字符串的开头是这 3 个字符串之一，说明是一个合法的排序值
                if (in_array($m[1], ['price', 'sold_count', 'rating'])) {
                    // 根据传入的排序值来构造排序参数
                    $builder->orderBy($m[1], $m[2]);
                }
            }
        }

        $products = $builder->paginate(16);

        return view('products.index', ['products' => $products]);
    }
```

这里详细解释一下模糊搜索那段代码，我们这里的实现是先用`$builder->where()`传入一个匿名函数，然后才在这个匿名函数里面再去添加`like`搜索，这样做目的是在查询条件的两边加上`()`，也就是说最终执行的 SQL 语句类似`select * from products where on_sale = 1 and ( title like xxx or description like xxx )`，试想如果我们直接用下面的代码：

```
$like = '%'.$search.'%';
$builder->where('title', 'like', $like)
    ->orWhere('description', 'like', $like)
    ->orWhereHas('skus', function ($query) use ($like) {
        $query->where('title', 'like', $like)
            ->orWhere('description', 'like', $like);
    });
```

那么生成的 SQL 就会变成`select * from products where on_sale = 1 and title like xxx or description like xxx`，这个 SQL 会把`on_sale = 0`但`description`包含搜索词的商品也搜索出来，这不符合我们的期望。

## 3. 测试

接下来我们测试一下，访问商品列表页，复制任意一个商品的名称，比如我这边的`dicta`

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/s06IU4RreD.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/s06IU4RreD.png!large)

粘贴到搜索框中然后点击`搜索`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/mI0i3b1sPX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/mI0i3b1sPX.png!large)

可以看到搜索成功，但是我们的搜索框变成了空。

## 4. 优化

为了实现保留用户的搜索内容，需要我们在控制器把用户的搜索和排序参数传到模板文件中：

_app/Http/Controllers/ProductsController.php_

```
 public function index(Request $request)
    {
        .
        .
        .
        return view('products.index', [
            'products' => $products,
            'filters'  => [
                'search' => $search,
                'order'  => $order,
            ],
        ]);
    }
```

接下来我们要通过 JS 的方式来把这两个变量放到表单中：

_resources/views/products/index.blade.php_

```
.
.
.
@section('scriptsAfterJs')
  <script>
    var filters = {!! json_encode($filters) !!};
    $(document).ready(function () {
      $('.search-form input[name=search]').val(filters.search);
      $('.search-form select[name=order]').val(filters.order);
    })
  </script>
@endsection
```

这里的`var filters = {!! json_encode($filters) !!};`把控制器传过来的`$filters`变量变成一个 JSON 字符串，赋值给 JS 的`filters`变量。

然后直接刷新刚刚的页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/XmpDyXnaOG.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/XmpDyXnaOG.png!large)

再试试排序方式是否也能保留，先把搜索框清空，排序方式选择`价格从低到高`，然后点击`搜索`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/AKbJ6c4SMH.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/AKbJ6c4SMH.png!large)

但是我们发现通过页面下方的翻页组件进入下一页之后这两个框又复原了，原本在地址栏里的搜索和排序参数也丢失了：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/EKpVWGK6Zk.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/EKpVWGK6Zk.png!large)

我们可以通过把`$filters`变量传给分页组件来解决这个问题：

_resources/views/products/index.blade.php_

```
.
.
.
<div class="float-right">{{ $products->appends($filters)->render() }}</div>
.
.
.
```

`appends()`方法接受一个 Key-Value 形式的数组作为参数，在生成分页链接的时候会把这个数组格式化成查询参数。

再测试一下，回到商品列表页，排序方式选择`价格从低到高`，搜索之后通过页面下方的分页组件进入第二页：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/9hEVKkAYRj.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/9hEVKkAYRj.png!large)

结果符合预期。

现在每次选择排序方式之后都需要点搜索按钮才能生效，这个体验不是很好，我们可以通过监听下拉框的`change`事件来触发表单自动提交：

_resources/views/products/index.blade.php_

```
.
.
.
@section('scriptsAfterJs')
  <script>
    var filters = {!! json_encode($filters) !!};
    $(document).ready(function () {
      .
      .
      .
      $('.search-form select[name=order]').on('change', function() {
        $('.search-form').submit();
      });
    })
  </script>
@endsection
```

刷新页面看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/NLHKp4HZdA.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/22/5320/NLHKp4HZdA.gif!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "用户界面 - 商品筛选、排序"
```



