## 新增收货地址页面

接下来我们要做的是新增收货地址的页面。

在`UserAddressesController`中新增一个`create()`方法：

_app/Http/Controllers/UserAddressesController.php_

```
use App\Models\UserAddress;
.
.
.
    public function create()
    {
        return view('user_addresses.create_and_edit', ['address' => new UserAddress()]);
    }
.
.
.
```

> 由于新增页面和编辑页面比较类似，所以共用一个模板文件`create_and_edit`。

然后添加一个路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    .
    .
    .
    Route::get('user_addresses/create', 'UserAddressesController@create')->name('user_addresses.create');
});
```

## 1. 省市区联动组件

收货地址一个核心功能就是省市区三级联动下拉，这是一个比较复杂的前端操作，我们这里使用 Vue 来实现。

首先我们添加一个中国省市区的库：

```
$ yarn add china-area-data
```

这个 nodejs 库包含了（基本上）最新的中国省市区的数据，通过类似邮编的方式将上下级关联起来，具体数据结构可以参考这个[代码文件](https://github.com/airyland/china-area-data/blob/master/v3/data.js)

接下来创建一个 Vue 的组件

```
$ touch resources/js/components/SelectDistrict.js
```

_resources/js/components/SelectDistrict.js_

```
// 从刚刚安装的库中加载数据
const addressData = require('china-area-data/v3/data');
// 引入 lodash，lodash 是一个实用工具库，提供了很多常用的方法
import _ from 'lodash';

// 注册一个名为 select-district 的 Vue 组件
Vue.component('select-district', {
  // 定义组件的属性
  props: {
    // 用来初始化省市区的值，在编辑时会用到
    initValue: {
      type: Array, // 格式是数组
      default: () => ([]), // 默认是个空数组
    }
  },
  // 定义了这个组件内的数据
  data() {
    return {
      provinces: addressData['86'], // 省列表
      cities: {}, // 城市列表
      districts: {}, // 地区列表
      provinceId: '', // 当前选中的省
      cityId: '', // 当前选中的市
      districtId: '', // 当前选中的区
    };
  },
  // 定义观察器，对应属性变更时会触发对应的观察器函数
  watch: {
    // 当选择的省发生改变时触发
    provinceId(newVal) {
      if (!newVal) {
        this.cities = {};
        this.cityId = '';
        return;
      }
      // 将城市列表设为当前省下的城市
      this.cities = addressData[newVal];
      // 如果当前选中的城市不在当前省下，则将选中城市清空
      if (!this.cities[this.cityId]) {
        this.cityId = '';
      }
    },
    // 当选择的市发生改变时触发
    cityId(newVal) {
      if (!newVal) {
        this.districts = {};
        this.districtId = '';
        return;
      }
      // 将地区列表设为当前城市下的地区
      this.districts = addressData[newVal];
      // 如果当前选中的地区不在当前城市下，则将选中地区清空
      if (!this.districts[this.districtId]) {
        this.districtId = '';
      }
    },
    // 当选择的区发生改变时触发
    districtId() {
      // 触发一个名为 change 的 Vue 事件，事件的值就是当前选中的省市区名称，格式为数组
      this.$emit('change', [this.provinces[this.provinceId], this.cities[this.cityId], this.districts[this.districtId]]);
    },
  },
  // 组件初始化时会调用这个方法
  created() {
    this.setFromValue(this.initValue);
  },
  methods: {
    // 
    setFromValue(value) {
      // 过滤掉空值
      value = _.filter(value);
      // 如果数组长度为0，则将省清空（由于我们定义了观察器，会联动触发将城市和地区清空）
      if (value.length === 0) {
        this.provinceId = '';
        return;
      }
      // 从当前省列表中找到与数组第一个元素同名的项的索引
      const provinceId = _.findKey(this.provinces, o => o === value[0]);
      // 没找到，清空省的值
      if (!provinceId) {
        this.provinceId = '';
        return;
      }
      // 找到了，将当前省设置成对应的ID
      this.provinceId = provinceId;
      // 由于观察器的作用，这个时候城市列表已经变成了对应省的城市列表
      // 从当前城市列表找到与数组第二个元素同名的项的索引
      const cityId = _.findKey(addressData[provinceId], o => o === value[1]);
      // 没找到，清空城市的值
      if (!cityId) {
        this.cityId = '';
        return;
      }
      // 找到了，将当前城市设置成对应的ID
      this.cityId = cityId;
      // 由于观察器的作用，这个时候地区列表已经变成了对应城市的地区列表
      // 从当前地区列表找到与数组第三个元素同名的项的索引
      const districtId = _.findKey(addressData[cityId], o => o === value[2]);
      // 没找到，清空地区的值
      if (!districtId) {
        this.districtId = '';
        return;
      }
      // 找到了，将当前地区设置成对应的ID
      this.districtId = districtId;
    }
  }
});
```

最后在`app.js`中引入这个组件：

_resources/js/app.js_

```
.
.
.
// 此处需在引入 Vue 之后引入
require('./components/SelectDistrict');

const app = new Vue({
    el: '#app'
});
```

> 本小节涉及到较多的 Vue / JS 知识，本课程的核心是 Laravel / PHP 的高级应用，因此不会太过详细解释每一个 Vue / JS 相关的知识点，有兴趣深入学习 Vue.js 的同学推荐学习下 Summer 的实战课程 ——[《Vue.js 实战教程 - 基础篇》](https://learnku.com/laravel/t/12298/vuejs)，可以为你节省很多时间。

## 2. 模板页面

接下来我们把刚才的 Vue 组件放到模板中：

```
$ touch resources/views/user_addresses/create_and_edit.blade.php
```

_resources/views/user\_addresses/create\_and\_edit.blade.php_

```
@extends('layouts.app')
@section('title', '新增收货地址')

@section('content')
  <div class="row">
    <div class="col-md-10 offset-lg-1">
      <div class="card">
        <div class="card-header">
          <h2 class="text-center">
            新增收货地址
          </h2>
        </div>
        <div class="card-body">
          <form class="form-horizontal" role="form">
            <!-- inline-template 代表通过内联方式引入组件 -->
            <select-district inline-template>
              <div class="form-row">
                <label class="col-form-label col-sm-2 text-md-right">省市区</label>
                <div class="col-sm-3">
                  <select class="form-control" v-model="provinceId">
                    <option value="">选择省</option>
                    <option v-for="(name, id) in provinces" :value="id">@{{ name }}</option>
                  </select>
                </div>
                <div class="col-sm-3">
                  <select class="form-control" v-model="cityId">
                    <option value="">选择市</option>
                    <option v-for="(name, id) in cities" :value="id">@{{ name }}</option>
                  </select>
                </div>
                <div class="col-sm-3">
                  <select class="form-control" v-model="districtId">
                    <option value="">选择区</option>
                    <option v-for="(name, id) in districts" :value="id">@{{ name }}</option>
                  </select>
                </div>
              </div>
            </select-district>
          </form>
        </div>
      </div>
    </div>
  </div>
@endsection
```

注意看`<select-district inline-template>`这一行，`select-district`就是我们的组件名称，之前在 JS 代码里通过`Vue.component('select-district', { /****/ })`定义的。`inline-template`则代表被`<select-district inline-template></select-district>`包裹的代码会作为这个组件的模板，也称为`内联模板`。

> 注意：`$ npm run watch-poll`要一直保持运行

现在访问一下页面看看效果：[http://shop.test/user\_addresses/create](http://shop.test/user_addresses/create)

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/YVikNiKGck.gif?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/YVikNiKGck.gif?imageView2/2/w/1240/h/0)

接下来我们需要把省市区组件的数据放到表单里提交给后端，创建一个新的 Vue 组件来监听`select-district`的`change`事件：

```
$ touch resources/js/components/UserAddressesCreateAndEdit.js
```

_resources/js/components/UserAddressesCreateAndEdit.js_

```
// 注册一个名为 user-addresses-create-and-edit 的组件
Vue.component('user-addresses-create-and-edit', {
  // 组件的数据
  data() {
    return {
      province: '', // 省
      city: '', // 市
      district: '', // 区
    }
  },
  methods: {
    // 把参数 val 中的值保存到组件的数据中
    onDistrictChanged(val) {
      if (val.length === 3) {
        this.province = val[0];
        this.city = val[1];
        this.district = val[2];
      }
    }
  }
});
```

其中`onDistrictChanged()`方法将用于处理`select-district`组件抛出的`change`事件，把事件的数据复制到本组件中。

最后在`app.js`中引入这个组件：

_resources/js/app.js_

```
.
.
.
require('./components/SelectDistrict');
require('./components/UserAddressesCreateAndEdit');
.
.
.
```

下面把新组件也放到模板中：

_resources/views/user\_addresses/create\_and\_edit.blade.php_

```
@extends('layouts.app')
@section('title', '新增收货地址')

@section('content')
<div class="row">
<div class="col-md-10 offset-lg-1">
<div class="card">
  <div class="card-header">
    <h2 class="text-center">
      新增收货地址
    </h2>
  </div>
  <div class="card-body">
    <!-- 输出后端报错开始 -->
    @if (count($errors) > 0)
      <div class="alert alert-danger">
        <h4>有错误发生：</h4>
        <ul>
          @foreach ($errors->all() as $error)
            <li><i class="glyphicon glyphicon-remove"></i> {{ $error }}</li>
          @endforeach
        </ul>
      </div>
    @endif
    <!-- 输出后端报错结束 -->
    <!-- inline-template 代表通过内联方式引入组件 -->
    <user-addresses-create-and-edit inline-template>
      <form class="form-horizontal" role="form">
        <!-- 引入 csrf token 字段 -->
      {{ csrf_field() }}
      <!-- 注意这里多了 @change -->
        <select-district @change="onDistrictChanged" inline-template>
          <div class="form-group row">
            <label class="col-form-label col-sm-2 text-md-right">省市区</label>
            <div class="col-sm-3">
              <select class="form-control" v-model="provinceId">
                <option value="">选择省</option>
                <option v-for="(name, id) in provinces" :value="id">@{{ name }}</option>
              </select>
            </div>
            <div class="col-sm-3">
              <select class="form-control" v-model="cityId">
                <option value="">选择市</option>
                <option v-for="(name, id) in cities" :value="id">@{{ name }}</option>
              </select>
            </div>
            <div class="col-sm-3">
              <select class="form-control" v-model="districtId">
                <option value="">选择区</option>
                <option v-for="(name, id) in districts" :value="id">@{{ name }}</option>
              </select>
            </div>
          </div>
        </select-district>
        <!-- 插入了 3 个隐藏的字段 -->
        <!-- 通过 v-model 与 user-addresses-create-and-edit 组件里的值关联起来 -->
        <!-- 当组件中的值变化时，这里的值也会跟着变 -->
        <input type="hidden" name="province" v-model="province">
        <input type="hidden" name="city" v-model="city">
        <input type="hidden" name="district" v-model="district">
        <div class="form-group row">
          <label class="col-form-label text-md-right col-sm-2">详细地址</label>
          <div class="col-sm-9">
            <input type="text" class="form-control" name="address" value="{{ old('address', $address->address) }}">
          </div>
        </div>
        <div class="form-group row">
          <label class="col-form-label text-md-right col-sm-2">邮编</label>
          <div class="col-sm-9">
            <input type="text" class="form-control" name="zip" value="{{ old('zip', $address->zip) }}">
          </div>
        </div>
        <div class="form-group row">
          <label class="col-form-label text-md-right col-sm-2">姓名</label>
          <div class="col-sm-9">
            <input type="text" class="form-control" name="contact_name" value="{{ old('contact_name', $address->contact_name) }}">
          </div>
        </div>
        <div class="form-group row">
          <label class="col-form-label text-md-right col-sm-2">电话</label>
          <div class="col-sm-9">
            <input type="text" class="form-control" name="contact_phone" value="{{ old('contact_phone', $address->contact_phone) }}">
          </div>
        </div>
        <div class="form-group row text-center">
          <div class="col-12">
            <button type="submit" class="btn btn-primary">提交</button>
          </div>
        </div>
      </form>
    </user-addresses-create-and-edit>
  </div>
</div>
</div>
</div>
@endsection
```

代码解析：

* `<`
  `select-district @change="onDistrictChanged" inline-template`
  `>`
  可以看到比之前多了一个
  `@change="onDistrictChanged"`
  ，代表
  `select-district`
  的
  `change`
  事件将由
  `onDistrictChanged`
  这个方法来处理。
* 当用户选择了省市区之后，触发了
  `select-district`
  组件的
  `change`
  事件，被
  `user-addresses-create-and-edit`
  的
  `onDistrictChanged`
  捕获之后就会更新
  `user-addresses-create-and-edit`
  组件内的数据，然后就映射到 3 个隐藏的
  `input`
  中去。

再后面的代码就是很简单表单 Html，这里不再过多解释。

刷新一下页面看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/Vv4iha0pwa.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/Vv4iha0pwa.png!large)

## 3. 提交表单

接下来要做的就是保存用户提交的数据，根据[编码规范](https://learnku.com/docs/laravel-specification/5.5/form-validation)，我们需要在`Request`类中完成数据校验，我们先创建一个`Request`基类：

```
$ php artisan make:request Request
```

_app/Http/Requests/Request.php_

```
<?php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class Request extends FormRequest
{
    public function authorize()
    {
        return true;
    }
}
```

接下来创建`UserAddressRequest`类并继承`Request`基类：

```
$ php artisan make:request UserAddressRequest
```

_app/Http/Requests/UserAddressRequest.php_

```
<?php

namespace App\Http\Requests;

class UserAddressRequest extends Request
{
    public function rules()
    {
        return [
            'province'      => 'required',
            'city'          => 'required',
            'district'      => 'required',
            'address'       => 'required',
            'zip'           => 'required',
            'contact_name'  => 'required',
            'contact_phone' => 'required',
        ];
    }
}
```

接下来在控制器中创建对应的方法：

_app/Http/Controllers/UserAddressesController.php_

```
use App\Http\Requests\UserAddressRequest;
.
.
.
    public function store(UserAddressRequest $request)
    {
        $request->user()->addresses()->create($request->only([
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
.
.
.
```

* Laravel 会自动调用
  `UserAddressRequest`
  中的
  `rules()`
  方法获取校验规则来对用户提交的数据进行校验，因此这里我们不需要手动调用
  `$this-`
  `>`
  `validate()`
  方法。
* `$request-`
  `>`
  `user()`
  获取当前登录用户。
* `user()-`
  `>`
  `addresses()`
  获取当前用户与地址的
  **关联关系**
  （注意：这里并不是获取当前用户的地址列表）
* `addresses()-`
  `>`
  `create()`
  在关联关系里创建一个新的记录。
* `$request-`
  `>`
  `only()`
  通过白名单的方式从用户提交的数据里获取我们所需要的数据。
* `return redirect()-`
  `>`
  `route('user_addresses.index');`
  跳转回我们的地址列表页面。

然后创建对应路由：

_routes/web.php_

```
.
.
.
Route::group(['middleware' => ['auth', 'verified']], function() {
    Route::get('user_addresses', 'UserAddressesController@index')->name('user_addresses.index');
    Route::get('user_addresses/create', 'UserAddressesController@create')->name('user_addresses.create');
    Route::post('user_addresses', 'UserAddressesController@store')->name('user_addresses.store');
});
```

最后我们把这个路由放到前台表单的`action`属性里：

_resources/views/user\_addresses/create\_and\_edit.blade.php_

```
.
.
.
<user-addresses-create-and-edit inline-template>
    <form class="form-horizontal" role="form" action="{{ route('user_addresses.store') }}" method="post">
.
.
.
```

刷新页面之后，我们可以试一下直接提交表单：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/xR4CLGXOUa.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/xR4CLGXOUa.png?imageView2/2/w/1240/h/0)

可以看到`UserAddressRequest`类的验证规则是生效的。

但是这个提示语言都是英文，对用户并不友好，我们需要改成中文：

