---
title: 代码整洁之道——变量定义
---

作为一个软件开发者，应使自己编写的代码具有很好可读性、很好的代码整洁度。对后期维护有着事半功倍的效果，也使另外的开发者能快速的明白你的编码思维。好的代码能让别人看了心情愉悦。

引用美国童子军军规：让营地比你来时更干净

如果每次签入时，代码都比签出时干净，那么代码就不会腐坏。清理并不一定要花多少功夫，也许只是改革变量名，拆分一个有点过长的函数，消除一点点重复代码，清理一个嵌套ifyuju。

### 有意义的命名

#### 名副其实

变量、函数或类的名称应该已经答复了所有的大问题。它该告诉你，它为什么会存在，它做什么事，应该怎么用。

	int d; //消失的时间，以日计

	int elapsedTimeInDays;
	int daysSinceCreation;
	int daysSinceModification;
	int fileAgeInDays;

####  避免误导

	accountList; //变量中避免使用list，如果容器中并非真是个List，就会引起错误的判断。用accountGroup、bunchOfAccounts,甚至直接用accounts都会好一些。

#### 做有意义的区分

	void copyChars(char a1[],char a2[]){

	}

	-------------------------------------

	void copyChars(char source[],char destination[]){

	}

	getActiveAccount();

	getActiveAccounts();

	getActivceAccountInfo();

程序员怎么能知道该调用哪个函数呢？

如果缺少明确约定，变量 moneyAmount 、money等同，customerInfo、customer等同,accountData、account等同，theMessage、message等同。

#### 使用读得出来的名称

genymdhms (生成日期，年，月，日，时，分，秒)

	class DtaRcrd102{
		private Date genymdhms;
		private Date modymdhms;
		private final String pszqint = "102";
	}

	class Customer {
		private Date generationTimestamp;
		private Date modificationTimestamp;
		private final String recordId = "102";
	}

#### 使用可搜索的名称

	WORK_DAYS_PER_WEEK = 5

代码中避免使用数字，数字不方面查找，它可能是某些文件名或其他常量定义的一部分。如果该常量是个长数字，又被人错该过就会逃过搜索，从而造成错误。

#### 避免成员前缀

不必用m_前缀来表明成员变量。应当把类和函数做的足够小，消除对成员前缀的需要。

#### 接口与实现

你怎么来命名工厂和具体类？IShapeFactory和ShapeFactory吗？

前缀I被滥用到了说好听点是干扰，说难听点根本就是废话的程度。我不想让用户知道我给他们的是接口。我想让他们知道那个是ShapeFactory。如果接口和实现必选一个来编码的话，宁肯选择实现：ShapeFactoryImp 甚至是丑陋的CShapeFactory.

#### 避免思维映射

不能因为代码用有a和b，就要取名c。

#### 类名

类名和对象名应该是名词或名词短语，如Customer、Account和AddressParser，避免使用Manager、Processor这样的类名。

#### 方法名

方法名应当时动词或动词短语，如postPayment、deletePage或save。

属性访问器、修改器和断言应该根据其值命名，并在前缀加上get、set和is。

	string name = employee.getName();
	customer.setName("mike");
	if(paycheck.isPosted())

重载构造器时，使用描述了参数的静态工厂方法名。例如：

	Complex fulcrumPoint = Complex.FromRealNumber(23.0);

通常好于
	
	Complex fulcrumPoint = new Complex(23.0);

可以考虑将相应的构造器设置成privete。强制使用这种命名手段。

#### 每个概念对应一个词 

给每个抽象概念选一个词，并且一以贯之。例如，使用fetch、retrieve和get来个多个类中的同种方法命名。

#### 别用双关语

避免将同一个单词用于不同目的。同一术语用于不同概念，基本上就是双关语了。可能好多个类里面都会有add方法。只要这些add	方法的参数列表和返回值在语义上等价，就一切顺利。



























