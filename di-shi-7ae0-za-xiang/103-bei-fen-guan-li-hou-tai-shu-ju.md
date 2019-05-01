## 备份管理后台数据

我们在管理后台配置的菜单、权限都是保存在数据库里的，没有办法像代码文件一样提交到 Git 版本库中，因此我们需要把开发环境中配置好的后台信息导出成 SQL 文件，然后把 SQL 文件提交到 Git 版本库中。

## 1. 确定要备份的数据表

首先我们需要明确要备份哪些表里的数据，打开数据库管理工具，看看数据库里有哪些与管理后台相关的表：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/829ojLfEEH.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/829ojLfEEH.png?imageView2/2/w/1240/h/0)

红框中以`admin_`开头的表就是 Laravel-Admin 使用的表，从表名我们可以判断出各自的作用：

* `admin_menu`
  —— 管理后台的菜单；
* `admin_operation_log`
  —— 管理后台操作日志；
* `admin_permissions`
  —— 权限列表；
* `admin_role_menu`
  —— 角色与菜单的关联关系；
* `admin_role_permissions`
  —— 角色与权限的关联关系；
* `admin_role_users`
  —— 管理员与角色的关联关系；
* `admin_roles`
  —— 角色列表；
* `admin_user_permissions`
  —— 管理员与权限的关联关系；
* `admin_users`
  —— 管理员列表。

可以看出除了`admin_operation_log`操作日志表之外，别的表都是需要导出的。

## 2. 执行导出

因为这是一个一次性的工作，没有必要专门写代码来处理导入和导出，所以我们选择直接用`mysqldump`这个命令行程序来导出数据库中的数据，从成本上来说比较合适：

```
$ mysqldump -t laravel-shop admin_menu admin_permissions admin_role_menu admin_role_permissions admin_role_users admin_roles admin_user_permissions admin_users > database/admin.sql

$ mysqldump -t laravel-shop admin_menu admin_permissions admin_role_menu admin_role_permissions admin_role_users admin_roles admin_user_permissions admin_users -u root -p ee19c18f54> database/admin.sql
```

命令解析：

* `-t`
  选项代表不导出数据表结构，这些表的结构我们会通过 Laravel 的 migration 迁移文件来创建；
* `laravel-shop`
  代表我们要导出的数据库名称，后面则是要导出的表列表；
* `>`
  ` database/admin.sql`
  把导出的内容保存到
  `database/admin.sql`
  文件中。

> 在 Homestead 环境中我们执行 Mysql 相关的命令都不需要账号密码，因为 Homestead 都已经帮我们配置好了。在线上执行 Mysql 命令时则需要在命令行里通过 -u 和 -p 参数指明账号密码，如：`mysqldump -uroot -p123456 laravel-shop > database/admin.sql`

[![](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/j2zgiE94CL.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/j2zgiE94CL.png?imageView2/2/w/1240/h/0)

现在看看`database/admin.sql`文件的内容：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/WfjwAg36n2.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/WfjwAg36n2.png?imageView2/2/w/1240/h/0)

[![](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/TxKgEY9xa0.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/TxKgEY9xa0.png?imageView2/2/w/1240/h/0)

可以看到我们需要导出的数据都在里面。

## 3. 测试

接下来我们要测试一下这份导出的 SQL 能否正常使用。

首先我们用 Laravel 的`migrate:fresh`命令将数据库全部清空并重新执行 migrate：

```
$ php artisan migrate:fresh
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/n8qOwpduRz.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/n8qOwpduRz.png?imageView2/2/w/1240/h/0)

接着我们通过 Mysql 命令导入 SQL：

```
$ mysql laravel-shop < database/admin.sql
```

[![](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/AfokZaIWWs.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/AfokZaIWWs.png?imageView2/2/w/1240/h/0)

没有任何提示，我们到数据库管理软件里看看：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/4GSfqnNK5y.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/06/5320/4GSfqnNK5y.png?imageView2/2/w/1240/h/0)

可以看到数据已经导入成功了。

## Git 代码版本控制

接着让我们将这个 SQL 文件加入到版本控制中：

```
$ git add -A
$ git commit -m "管理后台 SQL 文件"
```



