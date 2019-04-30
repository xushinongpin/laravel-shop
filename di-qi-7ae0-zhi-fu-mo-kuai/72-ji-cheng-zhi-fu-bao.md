[![](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/YEKw607ATt.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/YEKw607ATt.png?imageView2/2/w/1240/h/0)

## 支付宝支付

本章节将要实现集成支付宝支付。

## 获取支付宝沙箱参数

支付宝有一个沙箱环境，可以让我们不需要拥有真实的商家账号就可以进行支付的开发测试。

首先访问[https://openhome.alipay.com/platform/appDaily.htm?tab=info](https://openhome.alipay.com/platform/appDaily.htm?tab=info)，用你的支付宝账号登录之后会看到如下界面：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/Vb9YEX3NXW.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/Vb9YEX3NXW.png?imageView2/2/w/1240/h/0)

点击`设置应用公钥`链接：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/rApVWqwFAe.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/rApVWqwFAe.png?imageView2/2/w/1240/h/0)

点击`设置应用公钥`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/bSLZx7mU3Z.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/bSLZx7mU3Z.png?imageView2/2/w/1240/h/0)

点击`查看密钥生成方法`链接之后会跳转到一篇文档，里面可以下载 RSA 密钥生成工具，请根据自己的系统下载对应的版本并打开：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/J6K9Qtq20u.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/J6K9Qtq20u.png?imageView2/2/w/1240/h/0)

如上图，`密钥格式`选择`PKCS1`，`密钥长度`选择`2048`，然后点击`生成密钥`按钮。生成完毕之后点击`商户应用公钥`右侧的`复制公钥`按钮，将其内容粘贴到刚刚网页上的框中：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/Rh2LgGG98x.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/Rh2LgGG98x.png?imageView2/2/w/1240/h/0)

点击`保存`按钮。

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/mJRCp2Pknm.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/mJRCp2Pknm.png?imageView2/2/w/1240/h/0)

再点击页面上的`查看支付宝公钥`链接：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/n16bbQoJPW.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/n16bbQoJPW.png?imageView2/2/w/1240/h/0)

将这一段公钥复制下来。

## 配置参数

接下来将这些参数放到配置文件中：

_config/pay.php_

```
'app_id' => '你在支付宝沙箱看到的appid',
'ali_public_key' => '支付宝沙箱显示的公钥',
'private_key' => '刚刚生成的私钥',
```

## 支付测试

接下来我们要试一下能否正常跳转到支付宝的支付界面，在路由文件中新增一个临时的路由：

_routes/web.php_

```
Route::get('alipay', function() {
    return app('alipay')->web([
        'out_trade_no' => time(),
        'total_amount' => '1',
        'subject' => 'test subject - 测试',
    ]);
});
```

然后访问[http://shop.test/alipay](http://shop.test/alipay)

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/ZB8mP8EjIy.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/ZB8mP8EjIy.png?imageView2/2/w/1240/h/0)

可以看到已经跳转到支付宝的界面，看 URL 是沙箱环境无误。沙箱的支付账户密码可以在刚刚沙箱界面左侧的`沙箱账号`获得：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/er31mySfV9.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/er31mySfV9.png?imageView2/2/w/1240/h/0)

在支付页面填入刚刚拿到的账号密码：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/9zriYwUF9B.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/9zriYwUF9B.png?imageView2/2/w/1240/h/0)

登录之后输入支付密码：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/Q8jZbueG5E.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/Q8jZbueG5E.png?imageView2/2/w/1240/h/0)

点击`确认付款`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/WretqUCRAX.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/19/5320/WretqUCRAX.png?imageView2/2/w/1240/h/0)

可以看到付款成功。

> 记得把刚刚添加的测试路由删除。

## Git 代码版本控制

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "集成支付宝"
```



