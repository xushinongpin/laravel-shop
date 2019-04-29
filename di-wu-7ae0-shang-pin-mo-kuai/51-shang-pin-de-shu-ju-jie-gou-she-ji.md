## 说明

商品是电商系统的核心，这一节我们将一起来设计『商品』的数据库结构，以及创建数据模型。

## SKU

在开发之前，我们需要先了解一下商品 SKU 的概念。

SKU = Stock Keeping Unit（库存量单位），也可以称为『单品』。对一种商品而言，当其品牌、型号、配置、等级、花色、包装容量、单位、生产日期、保质期、用途、价格、产地等属性中任一属性与其他商品存在不同时，可称为一个单品。

为了让大家更好地理解商品与商品 SKU 的关系，我们拿天猫上面的商品作为例子：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/16/5320/Y1lXSYijHR.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/16/5320/Y1lXSYijHR.png?imageView2/2/w/1240/h/0)

在这个例子中，商品就是`iPhone 8`，不同的版本、不同的存储容量所对应的具体型号就是这个商品的 SKU，比如`iPhone 8 - 无需合约版 - 红色 - 64G`就是一个 SKU，`iPhone 8 - 无需合约版 - 红色 - 256G`是另外一个 SKU，不同的 SKU 价格可能不一样，库存也不一样。

商家在管理库存的时候就是以 SKU 为单位来操作的，比如入库时会写清楚 “购入 iPhone 8 - 无需合约版 - 红色 - 256G 10 台，购入 iPhone 8 - 无需合约版 - 红色 - 64G 10 台” 而不是 “购入 iPhone 20 台”。

> 为了方便讲解，本课程不会实现像天猫这种多维度的 SKU 系统，我们只实现一个维度的 SKU。多维度的 SKU 只不过运营层面工作量会大一些，简而言之就是编辑需要录入更多的商品类型而已，这里的数据结构和技术方案，也是适用于多维度 SKU 的。

## 1. 数据库表设计

我们会有两个数据表：

* `products`
  表，产品信息表，对应数据模型 Product ；
* `product_skus`
  表，产品的 SKU 表，对应数据模型 ProductSku 。

## 2. 字段设计

接下来我们需要整理好`products`表和`product_skus`表的字段名称和类型：

`products`表：

| 字段名称 | 描述 | 类型 | 加索引缘由 |
| :--- | :--- | :--- | :--- |
| id | 自增长 ID | unsigned big int | 主键 |
| title | 商品名称 | varchar | 无 |
| description | 商品详情 | text | 无 |
| image | 商品封面图片文件路径 | varchar | 无 |
| on\_sale | 商品是否正在售卖 | tiny int, default 1 | 无 |
| rating | 商品平均评分 | float, default 5 | 无 |
| sold\_count | 销量 | unsigned int, default 0 | 无 |
| review\_count | 评价数量 | unsigned int, default 0 | 无 |
| price | SKU 最低价格 | decimal | 无 |

> 商品本身没有固定的价格，我们在商品表放置`price`字段的目的是方便用户搜索、排序。

`product_skus`表：

| 字段名称 | 描述 | 类型 | 加索引缘由 |
| :--- | :--- | :--- | :--- |
| id | 自增长 ID | unsigned big int | 主键 |
| title | SKU 名称 | varchar | 无 |
| description | SKU 描述 | varchar | 无 |
| price | SKU 价格 | decimal | 无 |
| stock | 库存 | unsigne int | 无 |
| product\_id | 所属商品 id | unsigne big int | 外键 |

电商项目中与钱相关的有小数点的字段一律使用`decimal`类型，而不是`float`和`double`，后面两种类型在做小数运算时有可能出现精度丢失的问题，这在电商系统里是绝对不允许出现的。

定义`decimal`字段时需要两个参数，一个是数值总的精度（整数位 + 小数位），另一个参数则是小数位。对于我们这个系统来说总精度 10、小数位精度 2 即可满足需求（约 1 亿）。

## 3. 创建模型

```
$ php artisan make:model Models/Product -mf
$ php artisan make:model Models/ProductSku -mf
```

根据上面整理好的字段修改数据库迁移文件：

_database/migrations/&lt; your\_date &gt;\_create\_products\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('products', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('title');
            $table->text('description');
            $table->string('image');
            $table->boolean('on_sale')->default(true);
            $table->float('rating')->default(5);
            $table->unsignedInteger('sold_count')->default(0);
            $table->unsignedInteger('review_count')->default(0);
            $table->decimal('price', 10, 2);
            $table->timestamps();
        });
    }
```

_database/migrations/&lt; your\_date &gt;\_create\_product\_skus\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('product_skus', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('title');
            $table->string('description');
            $table->decimal('price', 10, 2);
            $table->unsignedInteger('stock');
            $table->unsignedBigInteger('product_id');
            $table->foreign('product_id')->references('id')->on('products')->onDelete('cascade');
            $table->timestamps();
        });
    }
```

然后修改模型文件：

_app/Models/Product.php_

```
.
.
.
    protected $fillable = [
                    'title', 'description', 'image', 'on_sale', 
                    'rating', 'sold_count', 'review_count', 'price'
    ];
    protected $casts = [
        'on_sale' => 'boolean', // on_sale 是一个布尔类型的字段
    ];
    // 与商品SKU关联
    public function skus()
    {
        return $this->hasMany(ProductSku::class);
    }
```

_app/Models/ProductSku.php_

```
.
.
.
    protected $fillable = ['title', 'description', 'price', 'stock'];

    public function product()
    {
        return $this->belongsTo(Product::class);
    }
```

然后执行迁移：

```
$ php artisan migrate
```

## Git 代码版本控制

接着让我们将这些变更加入到版本控制中：

```
$ git add -A
$ git commit -m "商品模型"
```



