---
title: 为什么说 Swift 比 Object-C 类型更安全？
date: 2017-08-15 10:01:25
---

### Type System 之static 较 dynamic更安全

编程语言大多都有自己的Type System, ObjC和Swift都有。

Type 就像自然语言里的名词、动词、介词等等，是一种避免代码表达错误的约束。Type不仅仅包括对int、float等这类Primitive type，对象的class type，还包括function、block不那么明显的type。

#### static VS dynamic

static：由编译器管理，在compile 阶段就做类型check。

dynamic：在runtime 阶段动态类型check（ Objective C 的 runtime 和 message 机制都属于dynamic）。

#### 举例如下

objc: int i = 0; i = @“test”; //compile warning 因为 type 的上下文信息是完整的，编译器可以做类型判断。

objc: id obj = [NSData new]; obj = [NSObject new]; 由于 id 可以指向任意对象类型，id 可以在不同的时间点里指向不同的类型，编译器此时无法根据类型信息作出判断，是否存在类型使用错误的。将类型check延迟到runtime中去检测，可能会存在调用实际对象中不存在的方法而crash。

swift: var i 编译器会提示：Type annotation missing in pattern，也就是缺少 type 信息。要声明一个变量，我们可以通过如下两种方式来提供 type 信息： swift var i = 0 //方式一，implicit typing var i: Int // var i:Int  方式二，explicit typing 方式一是通过赋值来做 type inference，方式二是通过显式的提供 type 信息。


显然，static type 比 dynamic type 更安全，编译器可以帮我们做类型检查，这就是为什么 Swift 比 Objective C 在 type safety 上更安全。

### Optional in Swift (Swift 比 Objective-C 更安全)

	User* user = [self getCurrentUser];
	*
	*
	*
	//继续业务

user可能是nil的情况下。在 Objective C 的 runtime 里，给 nil 对象发送消息也是安全的，这种安全只是表示不会 crash，但有可能原本应该执行的逻辑就没有继续下去了，从这一角度去看，nil 对象是对业务不安全的。

而且我们把这种 nil 的 case 下所造成的影响延迟到了 run time 。

<br/>
更合理的做法是在编译时就考虑 nil 这种 case。optional 正是为此而生，如果我们定义返回值为 optional，那么 optional 的使用方就一定要考虑值不存在的场景，如果漏处理了为 nil 的场景，就会编译器报错，这样不光不会 crash，而且对业务逻辑来说也是安全的。
<br/>
总结就是，当我们使用 optional 来写业务的时候，Swift 会强制我们去考虑 data 的各种可能性，这样写出来的函数，其逻辑就是完整的，全面的。