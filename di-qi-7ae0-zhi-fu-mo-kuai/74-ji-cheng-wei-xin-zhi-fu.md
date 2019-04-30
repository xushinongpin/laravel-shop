[[[![](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/dRVlDEiyZM.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/dRVlDEiyZM.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/dRVlDEiyZM.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201806/13/1/dRVlDEiyZM.png?imageView2/2/w/1240/h/0)

## 微信支付

上一节我们已经实现了支付宝支付，这一节我们要实现微信支付。

> 微信支付的开发需要有一个微信公众号并且开通了微信支付才能正常进行，申请微信支付需要有公司资质。对于手上没有支付商户号的同学，可以大概地看下，知悉整个操作流程，心里有个概念，等以后在工作中遇到此类需求时，也可以胸有成竹。

首先访问微信支付商户平台[https://pay.weixin.qq.com/](https://pay.weixin.qq.com/)。

## 1. 获取商户号

登录之后点击顶部导航的`产品中心`，再点击`扫码支付`：

[[[![](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/R7IJ6dZCGz.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/R7IJ6dZCGz.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/R7IJ6dZCGz.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/21/5320/R7IJ6dZCGz.png?imageView2/2/w/1240/h/0)

如果没有开通请先开通，开通之后点击页面上的`产品设置`按钮：

[[[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/LmRcCgR1jG.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/LmRcCgR1jG.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/LmRcCgR1jG.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/LmRcCgR1jG.png?imageView2/2/w/1240/h/0)

页面上会显示`商户号`，把这个商户号记下来，之后会用到。

[[[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/yQMtTwCVl3.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/yQMtTwCVl3.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/yQMtTwCVl3.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/yQMtTwCVl3.png?imageView2/2/w/1240/h/0)

## 2 下载 API 证书

接下来通过顶部导航切换到`账户中心`，点击左侧菜单`API 安全`，点击`下载证书`按钮：

[[[![](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/Tb0OnkvmEC.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/Tb0OnkvmEC.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/Tb0OnkvmEC.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/Tb0OnkvmEC.png?imageView2/2/w/1240/h/0)

再点击`下载`按钮：

[[[![](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/sRIOiEvi85.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/sRIOiEvi85.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/sRIOiEvi85.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/sRIOiEvi85.png?imageView2/2/w/1240/h/0)

会弹出提示要短信验证码和密码，输入之后证书就会被下载下来，应该有如下几个文件：

[[[![](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/3TBKIE4W9x.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/3TBKIE4W9x.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/3TBKIE4W9x.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/28/5320/3TBKIE4W9x.png?imageView2/2/w/1240/h/0)

我们只需要后面两个`.pem`后缀的文件。

接下来把这两个文件放入项目中，在项目中新建文件夹`resources/wechat_pay`：

```
$ mkdir -p resources/wechat_pay
```

然后将这两个文件复制到这个目录下。

## 3. 设置密钥

接着回到刚刚微信商户平台的页面，页面往下拉，会看到一个`设置密钥`按钮：

[[[![](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/WqhqDpQvqG.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/WqhqDpQvqG.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/WqhqDpQvqG.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201804/22/5320/WqhqDpQvqG.png?imageView2/2/w/1240/h/0)

> !!! 注意：如果这是一个已经在使用的支付号，一般是已经设置过密钥了，这里就不需要再次设置，修改密钥可能会影响已有的线上交易。

[[[![](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/St6OhvuTZZ.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/St6OhvuTZZ.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/St6OhvuTZZ.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/St6OhvuTZZ.png?imageView2/2/w/1240/h/0)

密钥可以通过[http://www.unit-conversion.info/texttools/random-string-generator/](http://www.unit-conversion.info/texttools/random-string-generator/)这个网站来生成：

[[[![](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/CWHTE7OgUz.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/CWHTE7OgUz.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/CWHTE7OgUz.png?imageView2/2/w/1240/h/0)](https://iocaffcdn.phphub.org/uploads/images/201805/25/5320/CWHTE7OgUz.png?imageView2/2/w/1240/h/0)

`Length`那个框填入`32`然后点击`Generate`按钮，下方的`Output`就会出现一个 32 个字符的随机字符串，把这个字符串复制下来。

接下来把这些配置参数写到我们的配置文件里

_config/pay.php_

```
'wechat' => [
        'app_id'      => 'wx*******',   // 公众号 app id
        'mch_id'      => '14*****',  // 第一步获取到的商户号
        'key'         => '******', // 刚刚设置的 API 密钥
        'cert_client' => resource_path('wechat_pay/apiclient_cert.pem'),
        'cert_key'    => resource_path('wechat_pay/apiclient_key.pem'),
        'log'         => [
            'file' => storage_path('logs/wechat_pay.log'),
        ],
    ],
```

## Git 代码版本控制

**注意：由于微信支付配置的是正式的参数，如果泄露将导致资金损失，所以千万不能把 config/pay.php 和 resources/wechat\_pay 目录下的文件提交到公共代码库中。可以用如下命令让 Git 忽略这些文件：**

```
$ git update-index --assume-unchanged config/pay.php
```

以及修改`.gitignored`文件：

_.gitignore_

```
.
.
.
/resources/wechat_pay
```

在正式的项目中由于不会提交到公开的代码仓库，因此不需要忽略这些文件。

接着让我们将这些文件加入到版本控制中：

```
$ git add -A
$ git commit -m "集成微信支付"
```



