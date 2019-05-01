## 配置后台权限

我们已经完成了后台所有的功能开发（用户管理、商品管理、订单管理、优惠券管理），现在需要给运营角色配置对应的权限。

## 1. 新增权限

由于用户管理权限我们之前已经创建过了，因此只需要创建 3 个权限。

点击后台左侧菜单`系统管理`-&gt;`权限`，点击`新增`按钮，按下图填写：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/6srhKio2ZF.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/6srhKio2ZF.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/xT8KrLmq9J.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/xT8KrLmq9J.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/5BU0SD6l30.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/5BU0SD6l30.png!large)

最终结果如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/veF9pCaHb9.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/veF9pCaHb9.png!large)

## 2. 给角色添加权限

接下来我们要把刚刚新增的权限添加给运营角色。

点击左侧菜单`系统管理`-&gt;`角色`，点击`运营`角色的编辑按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/ZAvXhgI96d.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/ZAvXhgI96d.png!large)

将`商品管理`等几个权限添加进去：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/wRANwUvnph.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/wRANwUvnph.png!large)

点击`提交`按钮。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/rDUcR1YNy1.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/rDUcR1YNy1.png!large)

## 3. 测试

接下来我们要测一下运营角色的用户能否正常使用后台的运营功能。

点击右上角头像，再点击`登出`按钮，然后使用`operator`作为用户名登录：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/ZhrE0YNSSs.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/ZhrE0YNSSs.png?imageView2/2/w/1240/h/0)

> 我们在 4.3 节的时候创建了这个账号，如果忘记密码可以登录管理员账号，在管理员管理页面修改这个账号的密码。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/UwbgUhoids.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/UwbgUhoids.png!large)

可以看到各个菜单，点击进去都可以正常访问访问：

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/UgMZwZuzGb.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/UgMZwZuzGb.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/bg9Up8DMxi.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/bg9Up8DMxi.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/LdD8KwiuHA.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/LdD8KwiuHA.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/p6TZNjb6D6.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/23/5320/p6TZNjb6D6.png!large)

## 结局

使用 Laravel-admin 构建后台的话，一般我们都会在大部分功能开发完成后，再来处理管理员权限，主要是为了避免疏忽造成权限分配错误。

