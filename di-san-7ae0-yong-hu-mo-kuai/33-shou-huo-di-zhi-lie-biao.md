## 收货地址

收货地址是电商网站必需的功能，本章节将要实现收货地址列表的展示。

## 1. 整理字段

在开始之前，我们需要先整理好`user_addresses`表的字段名称和类型：

| 字段名称 | 描述 | 类型 | 加索引缘由 |
| :--- | :--- | :--- | :--- |
| id | 自增长 ID | unsigned big int | 主键 |
| user\_id | 该地址所属的用户 | unsigned big int | 外键 |
| province | 省 | varchar | 无 |
| city | 市 | varchar | 无 |
| district | 区 | varchar | 无 |
| address | 具体地址 | varchar | 无 |
| zip | 邮编 | unsigned int | 无 |
| contact\_name | 联系人姓名 | varchar | 无 |
| contact\_phone | 联系人电话 | varchar | 无 |
| last\_used\_at | 最后一次使用时间 | datetime null | 无 |

## 2. 创建模型

通过`make:model`命令来创建一个新的模型：

```
$ php artisan make:model Models/UserAddress -fm
```

`-fm`参数代表同时生成`factory`工厂文件和`migration`数据库迁移文件

我们先根据上面整理出来的字段编写迁移文件：

_database/migrations/&lt; your\_date &gt;\_create\_user\_addresses\_table.php_

```
.
.
.
    public function up()
    {
        Schema::create('user_addresses', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->unsignedBigInteger('user_id');
            $table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
            $table->string('province');
            $table->string('city');
            $table->string('district');
            $table->string('address');
            $table->unsignedInteger('zip');
            $table->string('contact_name');
            $table->string('contact_phone');
            $table->dateTime('last_used_at')->nullable();
            $table->timestamps();
        });
    }
.
.
.
```

> 注：从 Laravel 5.8 开始，默认的自增 ID 的字段类型从原本的 int 改成了 big int，相应的 migration 代码应该使用`bigIncrements()`方法，定义对应自增 ID 的外键时也需要使用`unsignedBigInteger()`方法。

然后修改模型文件：

_app/Models/UserAddress.php_

```
.
.
.
    protected $fillable = [
        'province',
        'city',
        'district',
        'address',
        'zip',
        'contact_name',
        'contact_phone',
        'last_used_at',
    ];
    protected $dates = ['last_used_at'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function getFullAddressAttribute()
    {
        return "{$this->province}{$this->city}{$this->district}{$this->address}";
    }
.
.
.
```

代码解析：

* `protected $dates = ['last_used_at'];`
  表示
  `last_used_at`
  字段是一个时间日期类型，在之后的代码中
  `$address-`
  `>`
  `last_used_at`
  返回的就是一个时间日期对象（确切说是
  `Carbon`
  对象，
  `Carbon`
  是 Laravel 默认使用的时间日期处理类）。
* `public function user()`
  与
  `User`
  模型关联，关联关系是一对多（一个
  `User`
  可以有多个
  `UserAddress`
  ，一个
  `UserAddress`
  只能属于一个
  `User`
  ）。
* `public function getFullAddressAttribute()`
  创建了一个访问器，在之后的代码里可以直接通过
  `$address-`
  `>`
  `full_address`
  来获取完整的地址，而不用每次都去拼接。

接下来在`User`模型中关联上`UserAddress`模型：

_app/Models/User.php_

```
.
.
.
    public function addresses()
    {
        return $this->hasMany(UserAddress::class);
    }
.
.
.
```

然后执行迁移：

```
$ php artisan migrate
```

## 3. 创建控制器

通过`make:controller`命令创建`UserAddressesController`控制器：

```
$ php artisan make:controller UserAddressesController
```

添加`index()`方法：

_app/Http/Controllers/UserAddressesController.php_

```
.
.
.
    public function index(Request $request)
    {
        return view('user_addresses.index', [
            'addresses' => $request->user()->addresses,
        ]);
    }
.
.
.
```

我们把当前用户下的所有地址作为变量`$addresses`注入到模板`user_addresses.index`中并渲染。

## 4. 创建模板

接下来我们要创建收货地址列表页面的模板：

```
$ mkdir -p resources/views/user_addresses
$ touch resources/views/user_addresses/index.blade.php
```

_resources/views/user\_addresses/index.blade.php_

```
@extends('layouts.app')
@section('title', '收货地址列表')

@section('content')
  <div class="row">
    <div class="col-md-10 offset-md-1">
      <div class="card panel-default">
        <div class="card-header">收货地址列表</div>
        <div class="card-body">
          <table class="table table-bordered table-striped">
            <thead>
            <tr>
              <th>收货人</th>
              <th>地址</th>
              <th>邮编</th>
              <th>电话</th>
              <th>操作</th>
            </tr>
            </thead>
            <tbody>
            @foreach($addresses as $address)
              <tr>
                <td>{{ $address->contact_name }}</td>
                <td>{{ $address->full_address }}</td>
                <td>{{ $address->zip }}</td>
                <td>{{ $address->contact_phone }}</td>
                <td>
                  <button class="btn btn-primary">修改</button>
                  <button class="btn btn-danger">删除</button>
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

## 5. 创建路由

_routes/web.php_

```
.
.
.
// auth 中间件代表需要登录，verified中间件代表需要经过邮箱验证
Route::group(['middleware' => ['auth', 'verified']], function() {
    Route::get('user_addresses', 'UserAddressesController@index')->name('user_addresses.index');
});
```

在浏览器中访问：[http://shop.test/user\_addresses](http://shop.test/user_addresses)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/mGTyNWKs1f.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/mGTyNWKs1f.png!large)

现在数据库里是空的，我们可以通过工厂文件来自动生成收货地址假数据。

## 6. 工厂文件

在编写工厂文件之前我们需要先修改一项配置，因为`factory`工厂文件会使用`faker`来自动生成字段的内容，而默认情况下生成的数据都是是英文的，我们需要修改成中文：

_config/app.php_

```
.
.
.
'faker_locale' => 'zh_CN',
.
.
.
```

接下来编辑工厂文件：

_database/factories/UserAddressFactory.php_

```
<?php

use App\Models\UserAddress;
use Faker\Generator as Faker;

$factory->define(UserAddress::class, function (Faker $faker) {
    $addresses = [
        ["北京市", "市辖区", "东城区"],
        ["河北省", "石家庄市", "长安区"],
        ["江苏省", "南京市", "浦口区"],
        ["江苏省", "苏州市", "相城区"],
        ["广东省", "深圳市", "福田区"],
    ];
    $address   = $faker->randomElement($addresses);

    return [
        'province'      => $address[0],
        'city'          => $address[1],
        'district'      => $address[2],
        'address'       => sprintf('第%d街道第%d号', $faker->randomNumber(2), $faker->randomNumber(3)),
        'zip'           => $faker->postcode,
        'contact_name'  => $faker->name,
        'contact_phone' => $faker->phoneNumber,
    ];
});
```

我们预先设置了一批省市区，通过`randomElement()`方法随机取出一个。

下面我们在 tinker 里测试一下刚创建的工厂文件：

```
$ php artisan tinker
```

```
>>> factory(App\Models\UserAddress::class)->make()
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/Kpb6Y5lH6x.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/Kpb6Y5lH6x.png?imageView2/2/w/1240/h/0)

`factory()->make()`方法只是创建了`UserAddress`对象，但并没有保存到数据库中，我们可以用`create()`方法来存入数据库：

```
>>> factory(App\Models\UserAddress::class, 3)->create(['user_id' => 1])
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/lAb5f8sQN6.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/lAb5f8sQN6.png?imageView2/2/w/1240/h/0)

`factory(App\Models\UserAddress::class, 3)`代表创建 3 个`UserAddress`对象。

`create()`方法可以接受一个数组参数，数组中的数据会作为字段的值保存到数据库中，这里我们将刚刚创建的 3 个地址的`user_id`字段设为了`1`，也就是把这 3 个地址与`id`为`1`的用户关联了起来，也就是我们注册的第一个用户。

现在我们再访问[http://shop.test/user\_addresses](http://shop.test/user_addresses)：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/LGb79ruvPQ.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/LGb79ruvPQ.png!large)

地址列表页面就做好了，现在我们在菜单栏添加一下入口：

_resources/views/layouts/\_header.blade.php_

```
.
.
.
  <div class="dropdown-menu" aria-labelledby="navbarDropdown">
    <a href="{{ route('user_addresses.index') }}" class="dropdown-item">收货地址</a>
    .
    .
    .
  </div>
.
.
.
```

刷新页面，点击右上角的头像看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/jKnWjCFsMB.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/jKnWjCFsMB.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "收货地址列表"
```



