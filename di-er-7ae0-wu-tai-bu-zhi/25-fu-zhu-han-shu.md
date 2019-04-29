## 辅助函数

Laravel 提供了很多 辅助函数，有时候我们也需要创建自己的辅助函数。

我们把所有的『自定义辅助函数』存放于`bootstrap/helpers.php`文件中，创建这个文件，并且放入如下内容：

```
<?php
function test_helper() {
    return 'OK';
}
```

这个时候在 Vagrant 中进入 tinker

> tinker 是 Laravel 内置的一个交互式控制台，可以让我们很方便地调试 PHP 代码。

```
$ php artisan tinker
```

然后在 tinker 中执行我们刚刚添加的`test_helper`函数：

```
>>> test_helper()
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/KSuLNSjRW4.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/KSuLNSjRW4.png?imageView2/2/w/1240/h/0)

报错找不到这个函数，这是因为我们还没有引入这个 helpers.php 文件，我们可以使用 composer 的 autoload 功能来自动引入：

> 注：按 ctrl + d 退出 tinker 程序。

打开`composer.json`文件，并找到`autoload`段，将其修改为：

```
    "autoload": {
        "classmap": [
            "database/seeds",
            "database/factories"
        ],
        "psr-4": {
            "App\\": "app/"
        },
        "files": [
            "bootstrap/helpers.php"
        ]
    },
```

注意逗号不要多写或者漏写。

然后再在 Vagrant 中 项目根目录执行：

```
$ composer dumpautoload
```

然后再次进入 tinker 执行`test_helper`函数，可以看到有了正常返回：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/LbqQFVg5rG.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/LbqQFVg5rG.png?imageView2/2/w/1240/h/0)

`test_helper`函数只是我们用来测试 helpers.php 有没有被正常引用，因此可以把这个函数从 helpers.php 中删除。

## Git 代码版本控制

接着让我们将这些变更加入到版本控制中：

```
$ git add -A
$ git commit -m "辅助函数"
```



