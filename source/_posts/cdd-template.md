---
title: MrPeak CDD之模板脚本执行报错解决方法记录一下
date: 2017-07-15 10:01:25
---

参考：http://mrpeak.cn/blog/controller-demo/
![运行图](/images/cdd-template/1.png)

分为Controller，View，Presenter，Interactor，Adapter。Controller和View同MVC，Presenter配合Category语法负责业务逻辑和状态维护，Interactor做页面路由，Adapter处理UITableView的DataSource和Delegate。

结构目录：

![运行图](/images/cdd-template/2.png)

以上文件可使用模板脚本创建，避免重复劳动力.

执行模板脚本时报错：sed: RE error: illegal byte sequence

解决方法：在terminal 里执行：

export LC_COLLATE='C'
export LC_CTYPE='C'

再次执行模板脚本则成功。


![运行图](/images/cdd-template/3.png)

![运行图](/images/cdd-template/4.png)