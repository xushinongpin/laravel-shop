[![](https://iocaffcdn.phphub.org/uploads/images/201806/12/1/QOCFigH3Bv.jpeg?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/12/1/QOCFigH3Bv.jpeg?imageView2/2/w/1240/h/0)

## 命令行提示符

在本书的教授过程中，我将使用`$`符号来作为命令行提示符，如：

```
$ echo "Hello Laravel!"
Hello Laravel!
```

带有`$`符号的第一行代码指的是我们在命令行端口中输入的命令`echo "Hello Laravel!"`。`echo`是 Unix 系统中常用的输出命令，用于输出指定字符串。第二行的`Hello Laravel!`是运行命令后的输出信息。后面我们会使用这种风格来表示命令行的输入与输出，因此你在复制命令行的时候要注意不要把`$`和输出信息也复制进去了。

由于接下来的教程有时会在两个不同的机器环境上（本机环境和虚拟机环境，大部分情况下是在虚拟机环境上）来调用命令行输入，因此我们约定，在本机上调用的命令输入使用`>`符号，在虚拟机上调用的命令使用`$`符号。

以下命令行运行在虚拟机里：

```
$ echo "I am in VM!"
I am in VM!
```

以下命令行运行在**主机**上：

```
> echo "I am in Host Machine!"
I am in Host Machine!
```

## 相对文件路径

针对每个人不同的工作环境，本书将统一默认为项目的根目录，而不是项目在文件系统中的完整路径。

例如在我电脑中`UsersController.php`文件的完整路径为：

```
/Users/summer/Code/weibo/app/Http/Controllers/UsersController.php
```

但在本书中，文件名路径参照的是项目的根目录，显示如下：

```
app/Http/Controllers/UsersController.php
```

这样就能保证每个人看到的路径名称都一致了。

## 竖排 '...' 代码省略

最后，为了保持文章的篇幅简洁，我会将一些不必要的代码使用竖排的`.`来代替，你在复制本文代码块的时候，切记不要将`.`也一同复制进去。演示代码如下：

```
<?php

namespace App\Http\Controllers;
.
.
.
class UsersController extends Controller
{
    public function index()
    {
        $users = User::all();
        return view('users.index', compact('users'));
    }
}    
```

## 排版规范

此文档遵循[中文排版指南](https://github.com/sparanoid/chinese-copywriting-guidelines)规范，并在此之上遵守以下约定：

* 英文的左右保持一个空白，避免中英文字黏在一起；
* 使用全角标点符号；
* 严格遵循 Markdown 语法；
* 原文中的双引号（" "）请代换成中文的引号（「」符号怎么打出来见
  [这里](http://zhihu.com/question/19755746/answer/27233392)
  ）；
* 「
  `加亮`
  」和「
  **加粗**
  」和「\[链接\]\(\)」都需要在左右保持一个空格。



