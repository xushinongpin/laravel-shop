[![](https://iocaffcdn.phphub.org/uploads/images/201806/12/1/NZHvnXEsqI.jpeg?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/12/1/NZHvnXEsqI.jpeg?imageView2/2/w/1240/h/0)

## 说明

上一节我们分析了项目需求，在本节中，我们将简单做下项目的开发计划。很多时候，当开发团队开始启动一个项目时，区分功能模块的优先顺序尤其重要，否则你都不知道从哪里下手。这里我们使用一个简单的分析框架，来决策功能模块的开发优先级。你也可以使用其对大部分的 Web 项目进行模块开发的优先级分析。

## 1. 模块清单

首先，基于我们的需求分析，我们将系统拆分成如下几大模块：

* 用户模块
* 商品模块
* 订单模块
* 支付模块
* 优惠券模块
* 管理模块

## 2. 依赖关系

有了模块清单，接下来我们需要思考，他们之间的依赖关系是怎样的。在上面的功能清单中，『订单模块』依赖于『用户模块』和『商品模块』，『支付模块』和『优惠券模块』又依赖于『订单模块』。各个模块之间的依赖关系可以用下图来表示：

[![](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/mwSasI9sxg.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/07/5320/mwSasI9sxg.png?imageView2/2/w/1240/h/0)

上层的模块依赖于下层的模块，因此在开发过程中我们会优先构建下层的模块。

## 3. 开发顺序

所以我们各个模块开发的顺序如下：

1. 用户模块
2. 商品模块
3. 订单模块
4. 支付模块
5. 优惠券模块

『管理模块』是一个特殊的模块，既包含本身的逻辑（管理后台的权限控制等），又与其他业务模块都有关联，因此在开发过程中会与其他模块穿插开发。

## 4. MVP 产品

是的，作为工程师，我们不需要了解产品的方方面面，那是产品经理的工作。但是作为一位优秀的开发者，在开发项目时，对将要完成的产品 MVP 要了然于胸，MVP 是 Minimum Viable Product （最小化可行性产品）的简称。如何得出产品的 MVP 产品呢？可以先问这样的问题：

> 对于这个产品来讲，哪些功能是必不可缺的？

电商产品是一个用户购买商品的地方，产品存在的核心价值是『用户购买商品』，那首先需要用户、然后需要商品、购买需要付款。所以在我们的电商项目里，用户、商品、订单和支付模块都是必不可少的。

优惠券功能并不是购物流程中必备的一环，属于附加的功能，锦上添花的东西。我们在设计和开发项目时，应优先完成基础的功能，让流程能尽快跑起来，尽早交付，快速迭代。

Web 开发是个速度至上的领域，最小产品功能先上，测试的工作量也不会太大。不能憋大招，一个上线就是一大堆功能，复杂度增加的是无限的开发和调错时间，项目上线期限无尽延长。另一方面，用户能在最短时间内接触到产品，产品经理也可以尽快听到用户的反馈，及时调整产品战略，产品离成功会更进一步，这是一个多赢的方案。

这个思路也与敏捷开发的思路不谋而合：

> 敏捷开发即是以用户的需求进化为核心，采用迭代、循序渐进的方法进行软件开发。

## 结语

功能模块的开发优先级，我们已经有了，接下来就是要动手开始写代码了。

