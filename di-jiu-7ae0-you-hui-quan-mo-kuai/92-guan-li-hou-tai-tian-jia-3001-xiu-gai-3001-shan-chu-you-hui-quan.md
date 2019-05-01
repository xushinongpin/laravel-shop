## 添加优惠券

上一节我们完成了优惠券的设计和列表展示，接下来我们要实现在管理后台添加优惠券的功能。

## 1. 控制器

对于新增，我们只需要修改`create()`和`form()`方法即可：

_app/Admin/Controllers/CouponCodesController.php_

```
.
.
.
    public function create(Content $content)
    {
        return $content
            ->header('新增优惠券')
            ->body($this->form());
    }

    protected function form()
    {
        $form = new Form(new CouponCode);

        $form->display('id', 'ID');
        $form->text('name', '名称')->rules('required');
        $form->text('code', '优惠码')->rules('nullable|unique:coupon_codes');
        $form->radio('type', '类型')->options(CouponCode::$typeMap)->rules('required')->default(CouponCode::TYPE_FIXED);
        $form->text('value', '折扣')->rules(function ($form) {
            if (request()->input('type') === CouponCode::TYPE_PERCENT) {
                // 如果选择了百分比折扣类型，那么折扣范围只能是 1 ~ 99
                return 'required|numeric|between:1,99';
            } else {
                // 否则只要大等于 0.01 即可
                return 'required|numeric|min:0.01';
            }
        });
        $form->text('total', '总量')->rules('required|numeric|min:0');
        $form->text('min_amount', '最低金额')->rules('required|numeric|min:0');
        $form->datetime('not_before', '开始时间');
        $form->datetime('not_after', '结束时间');
        $form->radio('enabled', '启用')->options(['1' => '是', '0' => '否']);

        $form->saving(function (Form $form) {
            if (!$form->code) {
                $form->code = CouponCode::findAvailableCode();
            }
        });

        return $form;
    }
```

代码解析：

* 对于优惠码
  `code`
  字段，我们的第一个校验规则是
  `nullable`
  ，允许用户不填，不填的情况优惠码将由系统生成。
* 对于折扣
  `value`
  字段，我们的校验规则是一个匿名函数，当我们的校验规则比较复杂，或者需要根据用户提交的其他字段来判断时就可以用匿名函数的方式来定义校验规则。
* `$form-`
  `>`
  `saving()`
  方法用来注册一个事件处理器，在表单的数据被保存前会被触发，这里我们判断如果用户没有输入优惠码，就通过
  `findAvailableCode()`
  来自动生成。

注：为了绕过 Laravel-Admin 的一个[Bug](https://learnku.com/laravel/t/27467)，我们需要给 radio 类型的字段加上一个默认值。

## 2. 添加路由

_app/Admin/routes.php_

```
$router->post('coupon_codes', 'CouponCodesController@store');
$router->get('coupon_codes/create', 'CouponCodesController@create');
```

## 3. 测试

接下来我们来测试一下，先进入优惠券列表页面，点击页面上的`新增`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/9bkUdPsUoS.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/9bkUdPsUoS.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/kMxLf0tqBx.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/kMxLf0tqBx.png!large)

不填任何数据，直接提交看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/3lEtA1jtZT.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/3lEtA1jtZT.png!large)

优惠码这个字段没有报错，符合预期。

我们回到优惠券列表页面，随便找一个优惠券，将其优惠码复制下来：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/j5fs3kgqk8.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/j5fs3kgqk8.png!large)

再进入新增页面，将复制的优惠码填入，提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uwThMM2zFg.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/uwThMM2zFg.png!large)

看到提示`优惠码已存在`，符合预期。

然后我们来填一个正常的数据：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Si6xaBrYdd.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/Si6xaBrYdd.png!large)

点击`提交`按钮，可以看到回到了列表页面，并出现了刚刚新添加的优惠券：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qTEHXBOjyf.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qTEHXBOjyf.png!large)

## 修改优惠券

接下来我们要实现修改优惠券的功能。

## 1. 控制器

在`CouponCodesController`里添加`edit()`方法

_app/Admin/Controllers/CouponCodesController.php_

```
.
.
.
    public function edit($id, Content $content)
    {
        return $content
            ->header('编辑优惠券')
            ->body($this->form()->edit($id));
    }
```

## 2. 添加路由

_app/Admin/routes.php_

```
  $router->get('coupon_codes/{id}/edit', 'CouponCodesController@edit');
  $router->put('coupon_codes/{id}', 'CouponCodesController@update');
```

## 3. 测试

点击我们刚刚创建的优惠券右侧的编辑按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/9651FnQ2tn.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/9651FnQ2tn.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/gjO89kfS9Q.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/gjO89kfS9Q.png!large)

可以看到数据和我们之前填的都一致。

接下来我们给这个优惠券设置一下开始和结束时间看看：

> 开始时间请选择一个几天之后的时间点，我们之后会用到。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qYoldSjbtH.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/qYoldSjbtH.png!large)

然后点击保存：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/gcvrxlS7xt.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/gcvrxlS7xt.png!large)

发现校验不通过，提示优惠码已存在，这里我们需要调整一下优惠码的校验规则：

_app/Admin/Controllers/CouponCodesController.php_

```
.
.
.
    protected function form()
    {
        return Admin::form(CouponCode::class, function (Form $form) {
            .
            .
            .
            $form->text('code', '优惠码')->rules(function($form) {
                // 如果 $form->model()->id 不为空，代表是编辑操作
                if ($id = $form->model()->id) {
                    return 'nullable|unique:coupon_codes,code,'.$id.',id';
                } else {
                    return 'nullable|unique:coupon_codes';
                }
            });
            .
            .
            .
        });
    }
```

再次提交页面可以看到回到了列表页，再次进入编辑优惠券页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/B9hjmwTYue.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/B9hjmwTYue.png!large)

可以看到两个时间也都有了。

## 删除优惠券

接下来我们要实现优惠券的删除功能。由于 Laravel-Admin 已经帮我们内置了删除的方法，我们只需要配置一下路由即可：

_app/Admin/routes.php_

```
$router->delete('coupon_codes/{id}', 'CouponCodesController@destroy');
```

现在测一下，进入列表页面，选择一个之前我们批量生成的测试优惠券，点击删除按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/JvIGTIRw9w.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/JvIGTIRw9w.png!large)

顺着流程往下走：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/ybHlicpFRd.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/ybHlicpFRd.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/jwT9JdkM1N.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/jwT9JdkM1N.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/pUrIrcMzEu.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/pUrIrcMzEu.png!large)

发现页面上已经没有刚刚那条优惠券了。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 新增、修改、删除优惠券"
```



