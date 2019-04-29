## 基础布局

在教程开始之前，我们需要为我们的项目构建一个基础的页面布局，布局文件统一存放在`resources/views/layouts`文件夹中，布局涉及的文件如下：

* app.blade.php —— 主要布局文件，项目的所有页面都将继承于此页面；
* \_header.blade.php —— 布局的头部区域文件，负责顶部导航栏区块；
* \_footer.blade.php —— 布局的尾部区域文件，负责底部导航区块；

## 主要布局文件

我们先创建主要布局文件：`resources/views/layouts/app.blade.php`：

```
$ mkdir -p resources/views/layouts/
$ touch resources/views/layouts/app.blade.php
```

_resources/views/layouts/app.blade.php_

```
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <!-- CSRF Token -->
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>@yield('title', 'Laravel Shop') - Laravel 电商教程</title>
    <!-- 样式 -->
    <link href="{{ mix('css/app.css') }}" rel="stylesheet">
</head>
<body>
    <div id="app" class="{{ route_class() }}-page">
        @include('layouts._header')
        <div class="container">
            @yield('content')
        </div>
        @include('layouts._footer')
    </div>
    <!-- JS 脚本 -->
    <script src="{{ mix('js/app.js') }}"></script>
</body>
</html>
```

`route_class()`是我们自定义的辅助方法，我们还需要在`helpers.php`文件中添加此方法：

_bootstrap/helpers.php_

```
function route_class()
{
    return str_replace('.', '-', Route::currentRouteName());
}
```

此方法会将当前请求的路由名称转换为 CSS 类名称，作用是允许我们针对某个页面做页面样式定制。在后面的章节中会用到。

## 顶部导航

下面创建顶部导航模板：

```
$ touch resources/views/layouts/_header.blade.php
```

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
        <!-- Authentication Links -->
        <li class="nav-item"><a class="nav-link" href="#">登录</a></li>
        <li class="nav-item"><a class="nav-link" href="#">注册</a></li>
      </ul>
    </div>
  </div>
</nav>
```

注册登录链接我们将在后面章节中修改。

## 底部导航

创建文件：

```
$ touch resources/views/layouts/_footer.blade.php
```

_resources/views/layouts/\_footer.blade.php_

```
<footer class="footer">
  <div class="container">
    <p class="float-left">
      由 <a href="https://leo108.com" target="_blank">Leo</a> 设计和编码 <span style="color: #e27575;font-size: 14px;">❤</span>
    </p>

    <p class="float-right"><a href="mailto:name@email.com">联系我们</a></p>
  </div>
</footer>
```

## 首页展示

### 1. 创建控制器

我们将使用控制器`PagesController`来处理所有自定义页面的逻辑，并使用`root()`方法来处理首页的展示。

执行以下命令新建控制器：

```
$ php artisan make:controller PagesController
```

将会生成`app/Http/Controllers/PagesController.php`这个文件，我们里面新增`root()`方法：

_app/Http/Controllers/PagesController.php_

```
.
.
.
    public function root()
    {
        return view('pages.root');
    }
```

### 2. 视图

控制器`root()`方法中加载了视图`pages.root`，目前我们还没有此视图文件，前往创建：

```
$ mkdir -p resources/views/pages/
$ touch resources/views/pages/root.blade.php
```

_resources/views/pages/root.blade.php_

```
@extends('layouts.app')
@section('title', '首页')

@section('content')
  <h1>这里是首页</h1>
@stop
```

Laravel 自带了一个主页视图`welcome.blade.php`，既然我们已经自定义了主页视图，即可将废弃的主页视图删除：

```
$ rm resources/views/welcome.blade.php
```

### 3. 绑定路由

接下来绑定下路由，将`web.php`里的内容替换掉：

_routes/web.php_

```
<?php

Route::get('/', 'PagesController@root')->name('root');
```

### 4. 运行 Laravel Mix

Laravel Mix 一款前端任务自动化管理工具，使用了工作流的模式对制定好的任务依次执行。Mix 提供了简洁流畅的 API，让你能够为你的 Laravel 应用定义 Webpack 编译任务。Mix 支持许多常见的 CSS 与 JavaScript 预处理器，通过简单的调用，你可以轻松地管理前端资源。

使用 Mix 很简单，首先你需要使用以下命令安装 npm 依赖即可。我们将使用 Yarn 来安装依赖，在这之前，因为国内的网络原因，我们还需为 Yarn 配置安装加速：

```
$ yarn config set registry https://registry.npm.taobao.org
```

使用 Yarn 安装依赖：

```
$ SASS_BINARY_SITE=http://npm.taobao.org/mirrors/node-sass yarn
```

在`yarn`命令前添加`SASS_BINARY_SITE=http://npm.taobao.org/mirrors/node-sass`的目的是告诉`yarn`到淘宝的镜像去下载`node-sass`二进制文件。

然后我们还需要修改一下 Mix 的配置文件：

_webpack.mix.js_

```
.
.
.
mix.js('resources/js/app.js', 'public/js')
   .sass('resources/sass/app.scss', 'public/css')
   .version();
```

在末尾加了一个`version()`，使 Mix 每次生成的静态文件后面加上一个类似版本号的参数，避免浏览器缓存。

然后，运行以下命令即可：

```
$ npm install //没安装先安装
$ npm run watch-poll
```

`watch-poll`会在你的终端里持续运行，监控`resources`文件夹下的资源文件是否有发生改变。在`watch-poll`命令运行的情况下，一旦资源文件发生变化，Webpack 会自动重新编译。

> 注意：在后面的课程中，我们需要保证`npm run watch-poll`一直处在执行状态中。  
> Windows 用户如果遇到报错请参考[https://learnku.com/laravel/t/13277/in-learning-lessons-there-are-always-craters-recording-solutions](https://learnku.com/laravel/t/13277/in-learning-lessons-there-are-always-craters-recording-solutions)

正常运行的界面应类似：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/29/5320/AXclK5sMtg.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/29/5320/AXclK5sMtg.png?imageView2/2/w/1240/h/0)

现在访问[http://shop.test](http://shop.test/)看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/0F3npn9o1B.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/0F3npn9o1B.png!large)

## 优化页首与页脚

接下来我们来调整一下样式，样式代码在`resources/sass/app.scss`，是用 SASS 编写的，最终会由 Mix 来编译成 CSS。

_resources/sass/app.scss_

```
// Variables
@import 'variables';

// Bootstrap
@import '~bootstrap/scss/bootstrap';

/* universal */

body {
  font-family: Hiragino Sans GB, "Hiragino Sans GB", Helvetica, "Microsoft YaHei", Arial, sans-serif;
  font-size: 14px;
}

/* header */

.navbar-static-top {
  border-color: #e7e7e7;
  background-color: #fff;
  box-shadow: 0px 1px 11px 2px rgba(42, 42, 42, 0.1);
  border-top: 4px solid #00b5ad;
  border-bottom: 1px solid #e8e8e8;
  margin-bottom: 40px;
  margin-top: 0px;
}

/* Sticky footer styles */
html {
  position: relative;
  min-height: 100%;
}

body {
  /* Margin bottom by footer height */
  margin-bottom: 60px;
}

.footer {
  position: absolute;
  bottom: 0;
  width: 100%;
  /* Set the fixed height of the footer here */
  height: 60px;
  background-color: #000;

  .container {
    padding-right: 15px;
    padding-left: 15px;

    p {
      margin: 19px 0;
      color: #c1c1c1;

      a {
        color: inherit;
      }
    }
  }
}
```

再到浏览器中刷新页面看看效果：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/egHPLA7IP2.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/egHPLA7IP2.png!large)

至此，我们完成了基础页面结构的构建。

## Git 代码版本控制

在加入版本库之前我们先执行`git status`看看新增了哪些文件：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/KjOACw4t4A.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/KjOACw4t4A.png!large)

可以看到 Mix 还生成了`public/mix-manifest.json`这个文件，这也是不需要加入版本库的，在`.gitignore`中添加一行：

_.gitignore_

```
.
.
.
/public/mix-manifest.json
```

再次执行`git status`看看我们的变更是否生效：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/C21ie7fcgC.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/C21ie7fcgC.png!large)

确认没有问题，现在让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "基础布局"
```



