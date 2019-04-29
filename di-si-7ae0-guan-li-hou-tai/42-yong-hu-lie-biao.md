## 用户列表

用户管理是管理后台最常见的功能，本章节将实现管理后台用户列表页面。

## 1. 创建控制器

Laravel-Admin 的控制器创建方式与普通的控制器创建方式不太一样，要用`admin:make`来创建：

```
$ php artisan admin:make UsersController --model=App\\Models\\User
```

其中`--model=App\\Models\\User`代表新创建的这个控制器是要对`App\Models\User`这个模型做增删改查。

现在看一下自动生成的代码内容：

_app/Admin/Controllers/UsersController.php_

```
<?php

namespace App\Admin\Controllers;

use App\Models\User;
use App\Http\Controllers\Controller;
use Encore\Admin\Controllers\HasResourceActions;
use Encore\Admin\Form;
use Encore\Admin\Grid;
use Encore\Admin\Layout\Content;
use Encore\Admin\Show;

class UsersController extends Controller
{
    use HasResourceActions;

    /**
     * Index interface.
     *
     * @param Content $content
     * @return Content
     */
    public function index(Content $content)
    {
        return $content
            ->header('Index')
            ->description('description')
            ->body($this->grid());
    }

    /**
     * Show interface.
     *
     * @param mixed $id
     * @param Content $content
     * @return Content
     */
    public function show($id, Content $content)
    {
        return $content
            ->header('Detail')
            ->description('description')
            ->body($this->detail($id));
    }

    /**
     * Edit interface.
     *
     * @param mixed $id
     * @param Content $content
     * @return Content
     */
    public function edit($id, Content $content)
    {
        return $content
            ->header('Edit')
            ->description('description')
            ->body($this->form()->edit($id));
    }

    /**
     * Create interface.
     *
     * @param Content $content
     * @return Content
     */
    public function create(Content $content)
    {
        return $content
            ->header('Create')
            ->description('description')
            ->body($this->form());
    }

    /**
     * Make a grid builder.
     *
     * @return Grid
     */
    protected function grid()
    {
        $grid = new Grid(new User);

        $grid->id('Id');
        $grid->name('Name');
        $grid->email('Email');
        $grid->email_verified_at('Email verified at');
        $grid->password('Password');
        $grid->remember_token('Remember token');
        $grid->created_at('Created at');
        $grid->updated_at('Updated at');

        return $grid;
    }

    /**
     * Make a show builder.
     *
     * @param mixed $id
     * @return Show
     */
    protected function detail($id)
    {
        $show = new Show(User::findOrFail($id));

        $show->id('Id');
        $show->name('Name');
        $show->email('Email');
        $show->email_verified_at('Email verified at');
        $show->password('Password');
        $show->remember_token('Remember token');
        $show->created_at('Created at');
        $show->updated_at('Updated at');

        return $show;
    }

    /**
     * Make a form builder.
     *
     * @return Form
     */
    protected function form()
    {
        $form = new Form(new User);

        $form->text('name', 'Name');
        $form->email('email', 'Email');
        $form->datetime('email_verified_at', 'Email verified at')->default(date('Y-m-d H:i:s'));
        $form->password('password', 'Password');
        $form->text('remember_token', 'Remember token');

        return $form;
    }
}
```

其中`edit()`和`create()`用于编辑和创建用户，由于我们不会在管理后台去新增和编辑用户，所以可以把这两个方法删除，同时可以删除掉与之相关的`form()`方法。

`show()`方法用来展示用户详情页，通过调用`detail()`方法来决定要展示哪些字段，Laravel-Admin 会通过读取数据库自动把所有的字段都列出来，由于我们的用户表没有太多多余的字段，在列表页就可以直接展示，因此可以把`show()`和`detail()`方法也删掉。

`index()`方法用来展示用户列表，通过调用`grid()`方法来决定要展示哪些列，以及各个字段对应的名称，Laravel-Admin 会通过读取数据库自动把所有的字段都列出来。

现在我们来调整一下列表页面：

    .
    .
    .
        public function index(Content $content)
        {
            return $content
                ->header('用户列表')
                ->body($this->grid());
        }

        protected function grid()
        {
            $grid = new Grid(new User);

            // 创建一个列名为 ID 的列，内容是用户的 id 字段
            $grid->id('ID');

            // 创建一个列名为 用户名 的列，内容是用户的 name 字段。下面的 email() 和 created_at() 同理
            $grid->name('用户名');

            $grid->email('邮箱');

            $grid->email_verified_at('已验证邮箱')->display(function ($value) {
                return $value ? '是' : '否';
            });

            $grid->created_at('注册时间');

            // 不在页面显示 `新建` 按钮，因为我们不需要在后台新建用户
            $grid->disableCreateButton();

            $grid->actions(function ($actions) {
                // 不在每一行后面展示查看按钮
                $actions->disableView();
                // 不在每一行后面展示删除按钮
                $actions->disableDelete();
                // 不在每一行后面展示编辑按钮
                $actions->disableEdit();
            });

            $grid->tools(function ($tools) {
                // 禁用批量删除按钮
                $tools->batch(function ($batch) {
                    $batch->disableDelete();
                });
            });

            return $grid;
        }

由于我们并不关心用户验证邮箱的时间点，即`email_verified_at`字段的具体值，只需要知道用户是否已经验证过邮箱，所以我们调用了`display()`方法来更优雅地展示。`display()`方法接受一个匿名函数作为参数，在展示时会把对应字段值当成参数传给匿名函数，把匿名函数的返回值作为页面输出的内容。在这个例子里就是当`email_verified_at`有值时展示`是`，即验证过邮箱，否则展示`否`。

## 2. 添加路由

然后是路由，管理后台的路由文件路径是`app/Admin/routes.php`

_app/Admin/routes.php_

```
<?php

use Illuminate\Routing\Router;

Admin::registerAuthRoutes();

Route::group([
    'prefix'        => config('admin.route.prefix'),
    'namespace'     => config('admin.route.namespace'),
    'middleware'    => config('admin.route.middleware'),
], function (Router $router) {
    $router->get('/', 'HomeController@index');
    $router->get('users', 'UsersController@index');
});
```

> 如果没有特殊说明，本课程中所有在后台添加的路由均放在这个路由组中。

## 3. 添加菜单

我们可以直接在 Laravel-Admin 后台添加菜单：

打开左侧菜单的`系统管理`-&gt;`菜单`，进入菜单管理页面。

页面拉到下方的`新增`版块，标题填入`用户管理`，图表选择`fa-users`，路径填`/users`，点击提交。

[![](https://iocaffcdn.phphub.org/uploads/images/201804/09/5320/vR5e7j86qB.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/09/5320/vR5e7j86qB.png?imageView2/2/w/1240/h/0)

在上方的菜单树，将`用户管理`拖到`首页`下方，并点击`保存`按钮。

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/RJ9hkaCjVf.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/RJ9hkaCjVf.png?imageView2/2/w/1240/h/0)

刷新页面，可以看到左侧出现了`用户管理`的菜单。

[![](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/clp9Cy1nxl.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/01/5320/clp9Cy1nxl.png?imageView2/2/w/1240/h/0)

点击`用户管理`菜单查看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/kwoTuWZCfu.png!large "用户列表")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/kwoTuWZCfu.png!large)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 - 用户列表"
```



