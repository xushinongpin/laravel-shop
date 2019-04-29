## 管理后台权限设置

权限是管理后台一个很重要的部分，本章节将介绍如何在 Laravel-Admin 创建一个拥有用户管理权限的运营角色及创建对应的运营人员。

## 1. 新增权限

首先我们需要新建一个`用户管理`的权限。

进入管理后台，点击左侧菜单的`系统管理`-&gt;`权限`，点击`新增`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/IzWsWUopVl.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/IzWsWUopVl.png!large)

按照下图中的内容填入各个字段：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/x1cWdtOq84.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/x1cWdtOq84.png!large)

`标识`是用来标记权限的唯一标识，全局唯一。`名称`是这个权限的展示名称，要让人一眼看明白这个权限是做什么用的。如果用户访问的路由与`HTTP方法`和`HTTP路径`相匹配，则会检查该用户是否拥有本权限，我们这里只需要使用`/users*`即可匹配所有用户管理相关的路由。

然后点击`提交`按钮。

在权限页面就可以看到我们刚刚新添加的权限了：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/UWZCP0geoo.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/UWZCP0geoo.png!large)

## 2. 新增角色

接下来我们需要创建一个运营角色，并把刚刚创建的权限赋予该角色。

点击左侧菜单的`系统管理`-&gt;`角色`，点击`新增`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/cd7gEjnEPM.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/cd7gEjnEPM.png!large)

按照下图中的内容填入各个字段：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/HKxCukoDWF.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/HKxCukoDWF.png!large)

> 如果在选择权限的时候没有反应，可以试一下刷新页面再试试，应该是 Laravel-Admin 的 BUG。

`标识`是用来标记角色的唯一标识，全局唯一。`名称`是这个角色的展示名称。`权限`要点击选择`Login`/`User setting`/`Dashboard`/`用户管理`，前两个权限是必须的，否则该用户将无法登录后台和修改资料，第三个权限是管理后台的首页，如果没有这个权限，在登录的时候会报错。

然后点击`提交`按钮。

在角色页面就可以看到我们刚刚新添加的角色了：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/XzLVEANyaM.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/18/5320/XzLVEANyaM.png!large)

## 3. 新增管理后台用户

最后我们需要创建一个运营角色的用户。

点击左侧菜单的`系统管理`-&gt;`管理员`，点击`新增`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/OY9kqIllQu.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/OY9kqIllQu.png!large)

按照下图中的内容填入各个字段：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/JUGSJIIiQz.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/JUGSJIIiQz.png?imageView2/2/w/1240/h/0)

`用户名`即该用户的登录用户名，全局唯一。`名称`是这个用户的展示名称。`角色`选择我们刚刚创建的`运营`。`权限`留空。

> 我们这里通过角色来把用户和权限关联起来，而不是直接把用户和权限关联，这是因为运营的角色可能会有多个用户，假如运营角色的权限有变化，我们只需要修改运营角色的权限而不需要去修改每个运营用户的权限。

然后点击`提交`按钮。

在管理员页面就可以看到我们刚刚新添加的用户了：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/69s52noz11.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/69s52noz11.png!large)

## 测试

现在我们需要测试一下刚刚新添加的用户，点击右上角的头像，再点击`登出`按钮：

[![](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/uVkMIP2GFT.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201805/30/5320/uVkMIP2GFT.png?imageView2/2/w/1240/h/0)

用`operator`作为用户名登录：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/oYPI41Ysqp.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/oYPI41Ysqp.png!large)

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/dQ28LKFIdZ.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/dQ28LKFIdZ.png!large)

点击左侧菜单的用户管理，可以看到用户列表，符合预期：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/DuyOaEqu3T.png!large "权限设置")](https://iocaffcdn.phphub.org/uploads/images/201904/19/5320/DuyOaEqu3T.png!large)

> 由于本章节没有做任何代码上的改动，因此无需提交 Git。



上传图片显示不出来  请执行命令

```
php artisan storage:link
```



