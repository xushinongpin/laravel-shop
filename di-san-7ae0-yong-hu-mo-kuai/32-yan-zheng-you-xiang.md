## 验证邮箱

验证邮箱是各种系统很常见的一个功能。用户注册时，系统会往用户邮箱发送一封带有验证链接的邮件，用户点击该链接即可证明这个邮箱是真实存在并且被对应的用户所拥有。

## 1. 调整模型类

在开始开发之前，为了遵循[开发规范](https://learnku.com/docs/laravel-specification/5.5/data-model)，需要将`App\User`这个类移动到`App\Models\User`，全局搜索`App\User`：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/ryx0hFEW7D.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/08/5320/ryx0hFEW7D.png?imageView2/2/w/1240/h/0)

全部替换为`App\Models\User`。

创建`app/Models`目录，并将`app/User.php`移动到该目录下：

```
$ mkdir -p app/Models && mv app/User.php app/Models
```

最后修改`app/Models/User.php`的命名空间，将`namespace App;`改成`namespace App\Models;`

```
<?php

namespace App\Models;
.
.
.
```

从 Laravel 5.7 起，Laravel 自带了邮箱验证的相关字段和功能，我们只需要稍微调整即可。

首先调整一下 User 模型类：

_app/Models/User.php_

```
.
.
.
// 这里加上 MustVerifyEmail
class User extends Authenticatable implements MustVerifyEmail
.
.
.
```

由于 Laravel 在创建 User 类时就默认引入了`MustVerifyEmail`的命名空间，因此我们不需要再次引入。

## 2. 调整路由

接下来我们需要让 Laravel 启用与邮箱验证相关的路由（验证邮箱页面、重发验证邮件页面等），操作也很简单，只需要修改`web.php`文件：

_routes/web.php_

```
.
.
.
// 在之前的路由里加上一个 verify 参数
Auth::routes(['verify' => true]);
```

## 3. 验证邮箱中间件

Laravel 自带了一个名为`verified`的中间件，如果一个未验证邮箱的用户尝试访问一个配置了`verified`中间件的路由，Laravel 就会提示该用户邮箱未激活。

为了测试这个的中间件，我们修改一下之前的路由：

_routes/web.php_

```
// 在之前的路由后面配上中间件
Route::get('/', 'PagesController@root')->name('root')->middleware('verified');
.
.
.
```

先确保你已经处于登录状态，然后访问`http://shop.test/`，可以看到浏览器自动跳转到了`http://shop.test/email/verify`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/LyDAZ1XstX.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/LyDAZ1XstX.png!large)

实际上 Laravel 是通过 users 表中的`email_verified_at`字段来判断用户是否已经验证过邮箱，对于新注册的用户这个字段默认为`null`。现在我们尝试手动更改这个字段，将其改为当前时间点：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/wBTWxADz5q.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/wBTWxADz5q.png!large)

再次访问`http://shop.test/`

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/OFrwns6CAm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/OFrwns6CAm.png!large)

可以看到正确访问到了对应的页面。

接下来我们要测试一下发送验证邮件，所以先把`email_verified_at`字段改回`null`：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/NeH9MMRqZ0.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/NeH9MMRqZ0.png!large)

## 4. 配置 MailHog

在开始测试之前，我们需要先配置一下 MailHog。MailHog 是 Homestead 自带的一个组件，可以很方便地调试发送邮件。

我们在浏览器中访问[http://shop.test:8025](http://shop.test:8025/)，就可以看到 MailHog 的界面。

[![](https://iocaffcdn.phphub.org/uploads/images/201804/06/5320/zVqCcs9fCZ.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/06/5320/zVqCcs9fCZ.png?imageView2/2/w/1240/h/0)

我们修改一下`.env`文件，把 SMTP 服务器地址设置成`127.0.0.1`，端口设为`1025`：

_.env_

```
.
.
.
MAIL_DRIVER=smtp
MAIL_HOST=127.0.0.1
MAIL_PORT=1025
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_ENCRYPTION=null
.
.
.

//也可以自行在mailtrap.io这个网站注册邮箱测试
.
.
.
MAIL_DRIVER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=9dd22ef8f914b3
MAIL_PASSWORD=41a01d6350391c
MAIL_FROM_ADDRESS=from@example.com
MAIL_FROM_NAME=Example
.
.
.
```

这样在我们代码里发送的邮件都会在之前的面板里看到。

## 5. 发送验证邮件

现在再次访问`http://shop.test`，可以看到又跳转到了提示验证邮箱页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/ZTkTYQrpCC.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/ZTkTYQrpCC.png!large)

现在点击页面上的`click here to request another`链接：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/dhoTwYqj8T.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/dhoTwYqj8T.png!large)

可以看到提示验证邮件发送成功。现在去 Mailhog 的面板检查一下，访问[http://shop.test:8025](http://shop.test:8025/)：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/LueZRhJ3ud.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/LueZRhJ3ud.png!large)

可以看到收到了一封新邮件，点进去看看里面的内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/xS3ar0WIz7.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/xS3ar0WIz7.png!large)

点击`Verify Email Address`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/Kb9Z0G3ELa.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/Kb9Z0G3ELa.png!large)

可以看到跳转到了首页，并且不再提示邮箱没有验证。

现在再到数据库看看：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/eaP6oe8mOm.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/eaP6oe8mOm.png!large)

可以看到`email_verified_at`字段变成了我们验证邮箱的时间了。

## 6. 测试注册

接下来我们还需要测试一下新注册的用户是否会收到邮箱验证邮件，退出当前登录账户并进入注册页面：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/3M7z3I6qTE.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/3M7z3I6qTE.png!large)

填入一个新的邮箱地址并点击`Register`按钮。

现在再检查 MailHog 面板：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/S9jaJlZHMo.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/19/5320/S9jaJlZHMo.png!large)

可以看到又收到了一封邮件，并且收件人是新注册账号的邮箱。

经过上述测试，可以确定验证邮箱的功能能够正常运行，现在我们把之前添加的测试中间件去掉：

_routes/web.php_

```
Route::get('/', 'PagesController@root')->name('root');
.
.
.
```

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "邮箱验证功能"
```



