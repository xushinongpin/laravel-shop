## 管理后台

管理后台是各个项目必不可少的模块，从本章节开始，我们将逐步建立管理后台。本章将通过`encore/laravel-admin`来实现管理后台的基础框架。

## encore/laravel-admin 扩展包

`encore/laravel-admin`是一个可以快速构建后台管理的扩展包，它提供了页面组件和表单元素等功能，我们只需要使用很少的代码就实现功能完善的后台管理功能。

## 1. 安装

通过 Composer 来引入：

```
$ composer require encore/laravel-admin "1.6.11"
```

> 由于 Laravel-Admin 更新比较频繁，且小版本更新也有可能有较大的 API 变化，因此我们使用一个固定的版本。

然后按照官网文档指示，我们还需执行下面两个命令来完成安装：

```
$ php artisan vendor:publish --provider="Encore\Admin\AdminServiceProvider"
$ php artisan admin:install
```

第一个命令会将 Laravel-Admin 的一些文件发布到我们项目目录中，比如前端 JS/CSS 文件、配置文件等。第二个命令是执行数据库迁移、创建默认管理员账号、默认菜单、默认权限以及创建一些必要的目录。

我们可以通过`git status`来看看生成了哪些文件：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/20/5320/ou7ue1QPTF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/20/5320/ou7ue1QPTF.png!large)

* `app/Admin/`
  是用来放置管理后台的控制器和路由的目录；
* `config/admin.php`
  是
  `laravel-admin`
  的配置文件，我们一会儿会详细解释里面的内容；
* `database/migrations/2016_01_04_173148_create_admin_tables.php`
  用来创建与后台用户、角色、权限相关的数据库表；
* `public/vendor/`
  是
  `laravel-admin`
  会用到的一些前端库；
* `resources/lang/*`
  是语言文件，我们不需要除简体中文以外的语言，可以用下面命令删掉：

`$ rm -rf resources/lang/ar/ resources/lang/az/ resources/lang/en/admin.php resources/lang/es/ resources/lang/fr/ resources/lang/he/ resources/lang/ja/ resources/lang/nl/ resources/lang/pl/ resources/lang/pt/ resources/lang/ru/ resources/lang/tr/ resources/lang/zh-TW/ resources/lang/pt-BR/ resources/lang/fa/ resources/lang/id/ resources/lang/ms/ resources/lang/uk/`

## 2. 配置信息详解

请将以下内容替换并仔细阅读代码中的注释：

_config/admin.php_

```
<?php

return [

    /*
     * 站点标题
     */
    'name' => 'Laravel Shop',

    /*
     * 页面顶部 Logo
     */
    'logo' => '<b>Laravel</b> Shop',

    /*
     * 页面顶部小 Logo
     */
    'logo-mini' => '<b>LS</b>',

    /*
     * Laravel-Admin 启动文件路径
     */
    'bootstrap' => app_path('Admin/bootstrap.php'),

    /*
     * 路由配置
     */
    'route' => [
        // 路由前缀
        'prefix' => env('ADMIN_ROUTE_PREFIX', 'admin'),
        // 控制器命名空间前缀
        'namespace' => 'App\\Admin\\Controllers',
        // 默认中间件列表
        'middleware' => ['web', 'admin'],
    ],

    /*
     * Laravel-Admin 的安装目录
     */
    'directory' => app_path('Admin'),

    /*
     * Laravel-Admin 页面标题
     */
    'title' => 'Laravel Shop 管理后台',

    /*
     * 是否使用 https
     */
    'secure' => env('ADMIN_HTTPS', false),

    /*
     * Laravel-Admin 用户认证设置
     */
    'auth' => [

        'controller' => App\Admin\Controllers\AuthController::class,

        'guards' => [
            'admin' => [
                'driver'   => 'session',
                'provider' => 'admin',
            ],
        ],

        'providers' => [
            'admin' => [
                'driver' => 'eloquent',
                'model'  => Encore\Admin\Auth\Database\Administrator::class,
            ],
        ],

        // 是否展示 保持登录 选项
        'remember' => true,

        // 登录页面 URL
        'redirect_to' => 'auth/login',

        // 无需用户认证即可访问的地址
        'excepts' => [
            'auth/login',
            'auth/logout',
        ]
    ],

    /*
     * Laravel-Admin 文件上传设置
     */
    'upload' => [
        // 对应 filesystem.php 中的 disks
        'disk' => 'public',

        'directory' => [
            'image' => 'images',
            'file'  => 'files',
        ],
    ],

    /*
     * Laravel-Admin 数据库设置
     */
    'database' => [

        // 数据库连接名称，留空即可
        'connection' => '',

        // 管理员用户表及模型
        'users_table' => 'admin_users',
        'users_model' => Encore\Admin\Auth\Database\Administrator::class,

        // 角色表及模型
        'roles_table' => 'admin_roles',
        'roles_model' => Encore\Admin\Auth\Database\Role::class,

        // 权限表及模型
        'permissions_table' => 'admin_permissions',
        'permissions_model' => Encore\Admin\Auth\Database\Permission::class,

        // 菜单表及模型
        'menu_table' => 'admin_menu',
        'menu_model' => Encore\Admin\Auth\Database\Menu::class,

        // 多对多关联中间表
        'operation_log_table'    => 'admin_operation_log',
        'user_permissions_table' => 'admin_user_permissions',
        'role_users_table'       => 'admin_role_users',
        'role_permissions_table' => 'admin_role_permissions',
        'role_menu_table'        => 'admin_role_menu',
    ],

    /*
     * Laravel-Admin 操作日志设置
     */
    'operation_log' => [
        /*
         * 只记录以下类型的请求
         */
        'allowed_methods' => ['GET', 'HEAD', 'POST', 'PUT', 'DELETE', 'CONNECT', 'OPTIONS', 'TRACE', 'PATCH'],

        'enable' => true,

        /*
         * 不记操作日志的路由
         */
        'except' => [
            'admin/auth/logs*',
        ],
    ],

    /*
     * 地图组件提供商
     */
    'map_provider' => 'google',

    /*
     * 页面风格
     * @see https://adminlte.io/docs/2.4/layout
     */
    'skin' => 'skin-blue-light',

    /*
    |---------------------------------------------------------|
    |LAYOUT OPTIONS | fixed                                   |
    |               | layout-boxed                            |
    |               | layout-top-nav                          |
    |               | sidebar-collapse                        |
    |               | sidebar-mini                            |
    |---------------------------------------------------------|
     */
    'layout' => ['sidebar-mini', 'sidebar-collapse'],

    /*
     * 登录页背景图
     */
    'login_background_image' => '',

    /*
     * 显示版本
     */
    'show_version' => true,

    /*
     * 显示环境
     */
    'show_environment' => true,

    /*
     * 菜单绑定权限
     */
    'menu_bind_permission' => true,

    /*
     * 默认启用面包屑
     */
    'enable_default_breadcrumb' => true,

    /*
     * 扩展所在的目录.
     */
    'extension_dir' => app_path('Admin/Extensions'),

    /*
     * 扩展设置.
     */
    'extensions' => [

    ],
];
```

## 3. 效果展示

我们在配置文件中设定了路由前缀是`admin`，因此访问：[http://shop.test/admin](http://shop.test/admin)

> 默认的账号和密码都是`admin`

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/tEdT650SAK.png!large "安装 laravel-admin 扩展包")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/tEdT650SAK.png!large)

可以看到我们在配置里设置的标题生效了：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/C2oGQcrocv.png!large "安装 laravel-admin 扩展包")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/C2oGQcrocv.png!large)

## 4. 移除演示控制器

Laravel-Admin 在安装时默认创建了一个`ExampleController`，我们并不需要这个文件，可以直接删除：

```
$ rm -f app/Admin/Controllers/ExampleController.php
```

## 5. 菜单汉化

默认的菜单都是英文的，我们来汉化一下。

点击左侧菜单的`Admin`-&gt;`Menu`，点击`Index`菜单的编辑按钮

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/9AQq0jKMK4.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/9AQq0jKMK4.png?imageView2/2/w/1240/h/0)

将标题一栏改为`首页`：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/yzPJmRGqb4.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/yzPJmRGqb4.png?imageView2/2/w/1240/h/0)

点击`提交`按钮。

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/jXSSS0xEWw.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/jXSSS0xEWw.png?imageView2/2/w/1240/h/0)

同理，将`Admin`改为`系统管理`，`Users`改为`管理员`，`Roles`改为`角色`，`Permission`改为`权限`，`Menu`改为`菜单`，`Operation Log`改为`操作日志`。

最终效果如图：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/rww5N8CXyN.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/rww5N8CXyN.png?imageView2/2/w/1240/h/0)

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "初始化管理后台"
```



