## 系统安全

对于一个电商系统来说，安全性是一个很重要的指标，因为涉及到金钱交易，如果出现重大安全漏洞可能导致经济上的损失。

本章节将要介绍网站的常见漏洞类型及原理，以及对我们的电商系统的安全检查。

## 1. SQL 注入漏洞

SQL 注入是最古老、最流行同样也是危害最大的漏洞之一，这个漏洞的核心就一句话：将**未经过滤**的用户输入**拼接**到 SQL 语句中。

举个例子：

```
$product = DB::select("select * from products where id = '".$_GET['id']."'");
```

这个时候用户提交的`id`如果是`1 and 1 = 2`，那么最终被执行的 SQL 就是

```
select * from products where id = 1 and 1 = 2
```

很明显这个 SQL 查出来的结果必然为空，那么页面就会显示该商品不存在。

但仅仅是这样感觉好像没有什么危害，那这个时候攻击者提交的`id`参数变成了：

```
1 and exists (select * from admins where name = 'admin')
```

执行的 SQL 就变成了：

```
select * from products where id = 1 and exists (select * from admins where name = 'admin')
```

假如攻击者看到了正常的商品页面，那就说明这个网站存在一个名为`admin`的管理员。接着请求：

```
1 and exists (select * from admins where name = 'admin' and len(password) = 32)
```

这样根据页面是否展示商品信息就能判断出`admin`这个管理员在数据库中的密码是否为 32 位。这样就可以一步一步来验证出管理员的密码。现在有很多自动化的 SQL 注入软件，可以用很短的时间就把管理员的数据查询出来。

在 PHP 的项目里要避免 SQL 注入需要两个条件：

1. 使用 Prepared Statement + 参数绑定
2. 绝对不手动拼接 SQL

Prepared Statement 简单来说就是把要执行的 SQL 与 SQL 里的参数分开，还是以上面的查询商品为例，Prepared Statement 就是：

```
select * from products where id = ?
```

然后再传入参数：`1 and 1 = 1`，这个时候数据库会严格地去查找对应 ID =`1 and 1 = 1`的商品，而不是把`and 1 = 1`当做查询条件，所以即使`1 = 1`成立，数据库仍然返回空。

在 Laravel 里所有的 SQL 查询都是 Prepared Statement 模式，因此只要我们的代码中没有出现类似下方的代码，就不会存在 SQL 注入的风险：

```
$product = DB::select("select * from products where id = '".$_GET['id']."'");
```

## 2. XSS 漏洞

XSS 是与 SQL 注入齐名的漏洞，也可以用一句话来描述其核心：将**未经过滤**的用户输入**原样输出**到网页中。

例如用户把以下内容作为收货地址提交：

```
<script>alert(123)</script>
```

而如果我们在后台原样输出这个收货地址，就会触发网页 JS 代码，弹出一个 123 的提示框。

在我们的项目中，有多处用户输入会保存到数据库中，并在之后输出到了页面中：

* 用户昵称
* 收货地址的所有字段
* 下单时的备注字段
* 商品评价字段
* 申请退款理由

这些字段我们并没有做任何的过滤，现在核心是看看我们是否**原样输出**。

在 Laravel 中要输出文本有两种方式：`{{ $str }}`和`{!! $str !!}`，前者等同于`echo htmlspecialchars($str)`而后者则是`echo $str`，`htmlspecialchars()`函数默认会把`<>&"`这个 4 个字符分别转义成`<`、`&lgt;`、`&`和`"`，因此我们只要保证我们一直在用`{{ }}`来输出就没有问题。

所以排查方式很简单，搜索`resources/views`目录，看看我们有没有用到`{!! !!}`：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/zSsMFsExqW.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/zSsMFsExqW.png?imageView2/2/w/1240/h/0)

找到了两处，第一处是我们在实现商品搜索时，把用户的搜索条件以 JSON 格式输出到 JS 变量中，由于经过了一层 JSON 编码，所以是安全的；第二处是商品详情页输出商品详情，由于商品详情只能由我们的运营人员或者管理员来输入，因此不存在用户输入的风险。

## 3. CSRF 跨站请求伪造漏洞

CSRF 漏洞的危害程度不低于 XSS，但是远没有 XSS 那样出名。一句话描述就是：利用用户的身份认证信息在用户当前**已登录**的 Web 应用程序上执行非用户本意的操作。

举个例子，假如 Laravel China 退出登录的请求方式和路由是`Get /logout`，那么恶意用户只要在 LC 的帖子里插入一张图片，图片的 SRC 是`https://laravel-china.org/logout`，那么任何访问这个帖子的用户都会被强制退出，因为浏览器在请求这个 URL 之前并不知道这个 URL 是不是一张真的图片，那么就会带着当前用户的 Cookie 用 Get 方式去请求这个退出页面，也就导致当前用户被强制退出。

那么是不是把请求方式改成 Post 就没有问题了呢？确实无法通过图片的方式来触发，但是恶意用户可以在其他站点构造一个页面，内容如下：

```
<form action="https://laravel-china.org/logout" method="post" id="form"></form>
<script>document.gentElementById('form').submit();</script>
```

然后恶意用户诱导正常的用户去访问这个页面（比如这个恶意用户在 LC 的头条里分享了这个页面并取了一个很有诱惑的标题《教你如何实现一个千万并发的网站》，相信会有很多人访问），那么访问了这个页面的用户就会被强制退出 LC。

如果一个银行网站的转账功能有 CSRF 漏洞，那么恶意用户就可以通过 CSRF 漏洞来盗取其他用户的资金。

根据上面所说的原理，不难发现要避免 CSRF 攻击需要两个条件：

1. 敏感操作不能用 Get 请求方式；
2. 对于非 Get 的请求方式，需要校验请求中的
   `token`
   字段，这个字段值对每个用户每次登录都是不一样的。

对于 Laravel 项目来说已经内置了 CSRF 的防御手段，我们在写前端表单时都需要写一个`<input type="hidden" name="_token" value="{{ csrf_token() }}">`来提交 CSRF Token，因此我们的项目不会有 CSRF 攻击的风险。

## 4. 逻辑漏洞

对于电商系统，最常见的逻辑漏洞是添加负数个商品到购物车，如果没有做校验，并且有余额支付的渠道，那么恶意用户就可以通过下负数个商品的订单来增加自己的余额。

首先我们系统并没有提供余额支付的逻辑，因此不存在这个问题。

其次检查一下我们在添加商品到购物车和下单时的校验逻辑：

_app/Http/Requests/AddCartRequest.php:29_

```
   'amount' => ['required', 'integer', 'min:1'],
```

其中`min:1`代表提交的商品数最少为 1。

_app/Http/Requests/OrderRequest.php:39_

```
    'items.*.amount' => ['required', 'integer', 'min:1'],
```

同样要求`min:1`。

## 总结

由上面的这些漏洞及防御手段可以看到 Laravel 已经内置了绝大多数 Web 攻击的防御手段，只要我们牢记一些原则就能把绝大多数的安全风险扼杀在摇篮里。

