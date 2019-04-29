## 用户模块

用户模块是绝大多数网站都需要的功能，电商系统也不例外，在我们这个电商系统里用户即买家。本章节将要实现用户的登录和注册功能。

## 用户认证脚手架

Laravel 自带了用户认证功能，我们将利用此功能来快速构建我们的用户中心。

首先执行认证脚手架命令，生成代码：

```
$ php artisan make:auth
```

命令`make:auth`会询问我们是否要覆盖`app.blade.php`，因为我们在前面章节中已经自定义了『主要布局文件』——`app.blade.php`，所以此处输入`no`，如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/nzPqKLj8zM.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/nzPqKLj8zM.png?imageView2/2/w/1240/h/0)

使用`git status`来查看文件更改的状态：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/3TLML22AJ6.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/3TLML22AJ6.png?imageView2/2/w/1240/h/0)

打开`routes/web.php`查看修改了哪些内容：

_routes/web.php_

```
<?php

Route::get('/', 'PagesController@root')->name('root');
Auth::routes();

Route::get('/home', 'HomeController@index')->name('home');
```

`Auth::routes();`是 Laravel 的用户认证路由，这里不需要去修改。

再来看下面这一行：

```
 Route::get('/home', 'HomeController@index')->name('home');
```

我们已经有自己的主页了，不需要再次设置主页，直接删除即可。

同时删除`app/Http/Controllers/HomeController.php`和`resources/views/home.blade.php`两个文件：

```
$ rm -f app/Http/Controllers/HomeController.php resources/views/home.blade.php
```

由于我们删除了`/home`这个路由，因此需要把引用了这个路由的地方都修改掉：

修改`app/Http/Controllers/Auth/LoginController.php`、`app/Http/Controllers/Auth/RegisterController.php`、`app/Http/Controllers/Auth/VerificationController.php`和`app/Http/Controllers/Auth/ResetPasswordController.php`，将`$redirectTo`的值从`/home`改成`/`。

修改`app/Http/Middleware/RedirectIfAuthenticated.php`，将`redirect('/home')`修改为`redirect('/')`。

手动在浏览器导航栏里输入[http://shop.test/login](http://shop.test/login)，访问登录页面，看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/qi7zNxlkS1.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/qi7zNxlkS1.png!large)

## 顶部导航

接下来我们需要把顶部导航的登录和注册按钮指向真实的地址：

_resources/views/layouts/\_header.blade.php_

```
<nav class="navbar navbar-expand-lg navbar-light bg-light navbar-static-top">
    <div class="container">
        <!-- Branding Image -->
        <a class="navbar-brand " href="{{ url('/') }}">
            Laravel Shop
        </a>
        <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarSupportedContent" aria-controls="navbarSupportedContent" aria-expanded="false" aria-label="Toggle navigation">
            <span class="navbar-toggler-icon"></span>
        </button>

        <div class="collapse navbar-collapse" id="navbarSupportedContent">
            <!-- Left Side Of Navbar -->
            <ul class="navbar-nav mr-auto">

            </ul>

            <!-- Right Side Of Navbar -->
            <ul class="navbar-nav navbar-right">
                <!-- 登录注册链接开始 -->
                @guest
                    <li class="nav-item"><a class="nav-link" href="{{ route('login') }}">登录</a></li>
                    <li class="nav-item"><a class="nav-link" href="{{ route('register') }}">注册</a></li>
                @else
                    <li class="nav-item dropdown">
                        <a class="nav-link dropdown-toggle" href="#" id="navbarDropdown" role="button" data-toggle="dropdown" aria-haspopup="true" aria-expanded="false">
                            <img src="https://iocaffcdn.phphub.org/uploads/images/201709/20/1/PtDKbASVcz.png?imageView2/1/w/60/h/60" class="img-responsive img-circle" width="30px" height="30px">
                            {{ Auth::user()->name }}
                        </a>
                        <div class="dropdown-menu" aria-labelledby="navbarDropdown">
                            <a class="dropdown-item" id="logout" href="#"
                               onclick="event.preventDefault();document.getElementById('logout-form').submit();">退出登录</a>
                            <form id="logout-form" action="{{ route('logout') }}" method="POST" style="display: none;">
                                {{ csrf_field() }}
                            </form>
                        </div>
                    </li>
            @endguest
            <!-- 登录注册链接结束 -->
            </ul>
        </div>
    </div>
</nav>
```

刷新浏览器，将鼠标移到右上角的登录和注册链接，可看到已经指向了对应的地址。

[![](https://iocaffcdn.phphub.org/uploads/images/201803/27/5320/saExLnQB15.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201803/27/5320/saExLnQB15.png?imageView2/2/w/1240/h/0)

## 测试注册

先执行数据库迁移创建对应的数据库表结构：

```
$ php artisan migrate
```

然后通过顶部的注册链接访问注册页面，填入对应信息之后点击`Register`按钮

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/0nBdA1lH8I.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/0nBdA1lH8I.png!large)

可以看到注册成功并且已经登录为 leo 这个用户

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/lBmyNmq4ti.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/lBmyNmq4ti.png!large)

## 测试登录

点击右上角的下拉菜单中的『退出登录』按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/5CZssInZBG.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/5CZssInZBG.png!large)

点击右上角『登录』按钮，填入上一步注册时使用的邮箱和密码：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/axSLqK5gkb.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/axSLqK5gkb.png!large)

点击 Login 提交：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/dE0xMCe8i4.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/dE0xMCe8i4.png!large)

可以看到登录成功。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "注册与登录"
```



