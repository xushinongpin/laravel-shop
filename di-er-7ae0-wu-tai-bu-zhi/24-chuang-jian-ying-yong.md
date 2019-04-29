## 做好准备

由于我们接下来的开发都会在 Homestead 上进行，因此，在开始本章教程之前，请保证你的 Homestead 虚拟机已配置完成。

首先我们需要在 Homestead 的配置文件中做好站点配置和数据库配置：

_~/Homestead/Homestead.yaml_

```
sites:
    - map: shop.test
      to: /home/vagrant/Code/laravel-shop/public
      .
      .
      .
databases:
    - homestead
    - laravel-shop
```

我们这个项目将使用`laravel-shop`这个数据库。

然后还需要配置本机 hosts 文件，将`shop.test`指向 Homestead 虚拟机 IP，具体操作请参考[Laravel 开发环境部署](https://learnku.com/docs/laravel-development-environment/5.5)。

使用下面命令来启动和登录 Homestead：

```
> cd ~/Homestead && vagrant provision && vagrant up
> vagrant ssh
```

在虚拟机中进入 Code 文件夹：

```
$ cd ~/Code
```

> 注意：本书中因为虚拟机的存在，我们会有两个运行命令行的环境，一个是主机，另一个是 Homestead 虚拟机。我们会在命令的前面使用『命令行提示符』来区分主机和 Homestead。请记住以`>`开头的命令是运行在主机里，`$`开头的命令是运行在 Homestead 虚拟机里。详见[写作约定 - 命令行提示符](https://learnku.com/courses/laravel-intermediate-training/5.5/626/writing-convention#%E5%91%BD%E4%BB%A4%E8%A1%8C%E6%8F%90%E7%A4%BA%E7%AC%A6)。

## Composer 加速

在创建项目之前，我们先在虚拟机中运行以下命令来实现[Composer 安装加速](https://laravel-china.org/composer)：

```
$ composer config -g repo.packagist composer https://packagist.laravel-china.org
```

## 创建 Laravel-Shop 项目

下面让我们来使用 Composer 创建一个名为 Laravel-Shop 的应用，后面我们将基于这个应用做更多的功能完善：

```
$ cd ~/Code
$ composer create-project laravel/laravel laravel-shop --prefer-dist "5.8.*"
```

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/v9Dm4SRgR6.png!large "创建应用")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/v9Dm4SRgR6.png!large)

中间省略掉安装细节...

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/lSE5xduPEQ.png!large "创建应用")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/lSE5xduPEQ.png!large)

## 配置 .env

创建项目的时候 Laravel 也创建了一份默认的`.env`文件，我们只需要以下调整几个字段：

```
APP_NAME="Laravel Shop"
.
.
.
APP_URL=http://shop.test
.
.
.
DB_DATABASE=laravel-shop
```

> 如果环境变量的值中包含空格，需要用双引号将值包含起来

其他暂时不需要修改

## Git 代码版本控制

为了在接下来更好的追踪项目代码的更改，我们还需要将新建的 Laravel 项目纳入到 Git 版本管理中：

```
$ cd ~/Code/laravel-shop
$ git init
```

Laravel 默认的`.gitignore`文件并没有把`public/js`和`public/css`目录排除掉，前端编译后的 JS 和 CSS 文件会被放到这两个目录中，而编译后的文件是不需要加入到版本库的，因此先编辑`.gitignore`文件

_.gitignore_

```
.
.
.
/public/js
/public/css
```

然后再把这些文件加入到版本库中：

```
$ git add -A
$ git commit -m "初始化应用"
```



