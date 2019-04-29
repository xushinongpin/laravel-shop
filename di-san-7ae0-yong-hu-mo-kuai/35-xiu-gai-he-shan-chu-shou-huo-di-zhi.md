## 修改和删除收货地址

本章节我们将开发修改和删除收货地址功能，允许用户对已有的地址进行修改、删除。

## 1. 修改页面控制器和路由

在`UserAddressesController`类中新增`edit()`方法：

_app/Http/Controllers/UserAddressesController.php_

```
.
.
.
    public function edit(UserAddress $user_address)
    {
        return view('user_addresses.create_and_edit', ['address' => $user_address]);
    }
.
.
.
```

然后新增路由

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::get('user_addresses/{user_address}', 'UserAddressesController@edit')->name('user_addresses.edit');
});
```

> 注意：控制器的参数名`$user_address`必须和路由中的`{user_address}`一致才可以。

## 2. 模板页面

我们修改一下收货地址列表页面的`修改`按钮

_resources/views/user\_addresses/index.blade.php_

```
.
.
.
        <td>
          <a href="{{ route('user_addresses.edit', ['user_address' => $address->id]) }}" class="btn btn-primary">修改</a>
          <button class="btn btn-danger">删除</button>
        </td>
.
.
.
```

现在看下效果，访问地址列表页面[http://shop.test/user\_addresses](http://shop.test/user_addresses)，然后点击`修改`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/wIjWKSiyie.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/wIjWKSiyie.png?imageView2/2/w/1240/h/0)

可以看到部分字段已经自动填充为要编辑的数据，但是页面标题和省市区的展示不对。我们先把标题改一下：

### 修改页面标题

_resources/views/user\_addresses/create\_and\_edit.blade.php_

```
.
.
.
@section('title', ($address->id ? '修改': '新增') . '收货地址')
.
.
.
<h2 class="text-center">
  {{ $address->id ? '修改': '新增' }}收货地址
</h2>
.
.
.
```

刷新页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/y2vqydVya1.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/y2vqydVya1.png?imageView2/2/w/1240/h/0)

### 调整省市区组件

接下来是省市区，我们一开始在`select-district`组件里定义的`initValue`的属性现在可以派上用场了：

_resources/views/user\_addresses/create\_and\_edit.blade.php_

```
.
.
.
<select-district :init-value="{{ json_encode([$address->province, $address->city, $address->district]) }}" @change="onDistrictChanged" inline-template>
.
.
.
```

代码解析：

* 由于 Html 里的标签和属性不区分大小写，因此我们不能用
  `:initValue="xxxx"`
  来给组件传属性，这样传到 Vue 里就变成了
  `initvalue=xxx`
  。而 Vue 有个特性则是把 Html 属性名称里的
  `-{小写字母}`
  变成大写字母，也就是 Html 属性
  `init-value`
  对应 Vue 里的
  `initValue`
  属性。
* 在
  `init-value`
  前面有一个
  `:`
  ，这是 Vue 中
  `v-bind:`
  的缩写，展开来就是
  `v-bind:initValue="xxxx"`
  ，这用来指明后面的值是一个
  **表达式**
  而非
  **字符串**
  。我们在定义
  `initValue`
  属性的时候指定了这个属性是一个数组，所以我们传入的是一个由省市区名称组成的数组的 JSON 字符串，Vue 会把这个 JSON 字符串解析成数组赋值给组件内的
  `initValue`
  。

现在刷新页面看下效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/7nMlksdsAL.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/7nMlksdsAL.png?imageView2/2/w/1240/h/0)

可以看到省市区下拉框已经变成对应的选项了。

### 修改表单提交地址和方法

接下来我们需要修改一下表单的提交地址：

_resources/views/user\_addresses/create\_and\_edit.blade.php_

```
.
.
.
<user-addresses-create-and-edit inline-template>
  @if($address->id)
    <form class="form-horizontal" role="form" action="{{ route('user_addresses.update', ['user_address' => $address->id]) }}" method="post">
      {{ method_field('PUT') }}
  @else
    <form class="form-horizontal" role="form" action="{{ route('user_addresses.store') }}" method="post">
  @endif
    {{ csrf_field() }}
.
.
.
```

* 如果当前的
  `$address`
  变量有
  `id`
  属性，说明我们正在进行地址修改，需要把表单的
  `action`
  改成对应的 PUT 地址。
* `method_field('PUT')`
  会在页面中插入一个隐藏的
  `input`
  来告诉 Laravel 把这个表单的请求方式当成
  `PUT`
  。

刷新页面发现报错：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/07/5320/qQz8ZsiGBp.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/07/5320/qQz8ZsiGBp.png?imageView2/2/w/1240/h/0)

这是因为我们还没有定义处理修改逻辑的控制器和路由。

## 3. 处理修改的控制器和路由

在`UserAddressesController`中添加`update()`方法：

_app/Http/Controllers/UserAddressesController.php_

```
.
.
.
    public function update(UserAddress $user_address, UserAddressRequest $request)
    {
        $user_address->update($request->only([
            'province',
            'city',
            'district',
            'address',
            'zip',
            'contact_name',
            'contact_phone',
        ]));

        return redirect()->route('user_addresses.index');
    }
```

> 由于校验逻辑一样，我们直接使用之前的`UserAddressRequest`类做输入校验。

新增路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::put('user_addresses/{user_address}', 'UserAddressesController@update')->name('user_addresses.update');
});
```

刷新页面，试试看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/Xjt8o35r9j.gif?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/Xjt8o35r9j.gif?imageView2/2/w/1240/h/0)

## 4. 删除地址的控制器和路由

接下来我们要实现地址的删除功能。

在`UserAddressesController`中新增一个`destroy()`方法：

```
.
.
.
    public function destroy(UserAddress $user_address)
    {
        $user_address->delete();

        return redirect()->route('user_addresses.index');
    }
```

新增路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::delete('user_addresses/{user_address}', 'UserAddressesController@destroy')->name('user_addresses.destroy');
});
```

## 5. 删除按钮

我们需要把之前的假的删除按钮替换掉：

_resources/views/user\_addresses/index.blade.php_

```
.
.
.
  <a href="{{ route('user_addresses.edit', ['user_address' => $address->id]) }}" class="btn btn-primary">修改</a>
  <form action="{{ route('user_addresses.destroy', ['user_address' => $address->id]) }}" method="post" style="display: inline-block">
    {{ csrf_field() }}
    {{ method_field('DELETE') }}
    <button class="btn btn-danger" type="submit">删除</button>
  </form>
.
.
.
```

代码解析：

* `csrf_field()`
  在页面中插入一个隐藏的
  `input`
  ，包含了提交表单所需要的
  `csrf token`
  。
* `method_field('DELETE')`
  会在页面中插入一个隐藏的
  `input`
  来告诉 Laravel 把这个表单的请求方式当成
  `DELETE`
  。

进到地址列表页面，看下效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/20/5320/vZEwKG1kgZ.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/20/5320/vZEwKG1kgZ.gif!large)

## 6. 优化交互

虽然我们已经完成了删除功能，但是如果用户不小心点了删除按钮，对应的地址就立即被删除了，这样的用户体验很不好，所以我们需要加一个二次确认的操作。

首先通过 yarn 引入`sweetalert`这个库，`sweetalert`可以用来展示比较美观的弹出提示框：

```
$ yarn add sweetalert
```

然后在`bootstrap.js`引入这个库：

_resources/js/bootstrap.js_

```
require('sweetalert');
.
.
.
```

> 请确保`npm run watch-poll`在运行

然后要在这个页面上通过 JS 来调用`sweetalert`弹出二次确认提示框。

为了让页面的 Html 结构更合理，我们在`app.blade.php`中添加一个`@yield`命令：

_resources/views/layouts/app.blade.php_

```
.
.
.
<script src="{{ mix('js/app.js') }}"></script>
@yield('scriptsAfterJs')
.
.
.
```

然后就可以在模板中使用`@section`命令，使`@section`命令的内容出现在刚刚`@yield`命令所在的位置：

_resources/views/user\_addresses/index.blade.php_

```
.
.
.
<a href="{{ route('user_addresses.edit', ['user_address' => $address->id]) }}" class="btn btn-primary">修改</a>
<!-- 把之前删除按钮的表单替换成这个按钮，data-id 属性保存了这个地址的 id，在 js 里会用到 -->
<button class="btn btn-danger btn-del-address" type="button" data-id="{{ $address->id }}">删除</button>
.
.
.
@section('scriptsAfterJs')
<script>
$(document).ready(function() {
  // 删除按钮点击事件
  $('.btn-del-address').click(function() {
    // 获取按钮上 data-id 属性的值，也就是地址 ID
    var id = $(this).data('id');
    // 调用 sweetalert
    swal({
        title: "确认要删除该地址？",
        icon: "warning",
        buttons: ['取消', '确定'],
        dangerMode: true,
      })
    .then(function(willDelete) { // 用户点击按钮后会触发这个回调函数
      // 用户点击确定 willDelete 值为 true， 否则为 false
      // 用户点了取消，啥也不做
      if (!willDelete) {
        return;
      }
      // 调用删除接口，用 id 来拼接出请求的 url
      axios.delete('/user_addresses/' + id)
        .then(function () {
          // 请求成功之后重新加载页面
          location.reload();
        })
    });
  });
});
</script>
@endsection
```

删除接口的请求方式从表单提交改成了 AJAX 请求，因此我们还需调整一下删除接口的返回值，否则 AJAX 请求会有异常：

_app/Http/Controllers/UserAddressesController.php_

```
.
.
.
    public function destroy(UserAddress $user_address)
    {
        $user_address->delete();
        // 把之前的 redirect 改成返回空数组
        return [];
    }
```

现在刷新页面看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/20/5320/SnG11sGzPP.gif!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/20/5320/SnG11sGzPP.gif!large)

## 7. 检查权限

接下来我们要增加权限控制，只允许地址的拥有者来修改和删除地址，这里通过授权策略类（Policy）来实现权限控制：

```
$ php artisan make:policy UserAddressPolicy
```

新创建的 Policy 文件位于`app/Policies`目录下。

在`UserAddressPolicy`类中新建一个`own()`方法：

_app/Policies/UserAddressPolicy.php_

```
use App\Models\UserAddress;
.
.
.
    public function own(User $user, UserAddress $address)
    {
        return $address->user_id == $user->id;
    }
```

当`own()`方法返回`true`时代表当前登录用户可以修改对应的地址。

接下来还需要在`AuthServiceProvider`注册这个授权策略，从 Laravel 5.8 起，我们可以定义一个回调函数来让 Laravel 自己去寻找模型所对应的授权策略文件：

_app/Providers/AuthServiceProvider.php_

```
.
.
.
    public function boot()
    {
        $this->registerPolicies();

        // 使用 Gate::guessPolicyNamesUsing 方法来自定义策略文件的寻找逻辑
        Gate::guessPolicyNamesUsing(function ($class) {
            // class_basename 是 Laravel 提供的一个辅助函数，可以获取类的简短名称
            // 例如传入 \App\Models\User 会返回 User
            return '\\App\\Policies\\'.class_basename($class).'Policy';
        });
    }
.
.
.
```

最后在控制器中添加校验权限的代码：

_app/Http/Controllers/UserAddressesController.php_

```
.
.
.
    public function edit(UserAddress $user_address)
    {
        $this->authorize('own', $user_address);

        return view('user_addresses.create_and_edit', ['address' => $user_address]);
    }

    public function update(UserAddress $user_address, UserAddressRequest $request)
    {
        $this->authorize('own', $user_address);
        .
        .
        .
    }

    public function destroy(UserAddress $user_address)
    {
        $this->authorize('own', $user_address);
        .
        .
        .
    }
```

代码解析：

`authorize('own', $user_address)`方法会获取第二个参数`$user_address`的类名:`App\Models\UserAddress`，然后执行我们之前在`AuthServiceProvider`类中定义的自动寻找逻辑，在这里找到的类就是`App\Policies\UserAddressPolicy`，之后会实例化这个策略类，再调用名为`own()`方法，如果`own()`方法返回`false`则会抛出一个未授权的异常。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "修改和删除收货地址"
```



