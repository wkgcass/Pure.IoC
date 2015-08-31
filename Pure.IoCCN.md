click [here](https://github.com/wkgcass/Pure.IoC) to get English tutorial.

#Pure.IoC
Pure.IoC是一个轻量级基于类和注解的自动依赖注入框架。

>使用jdk 1.8  
>此框架依赖于[Style函数式工具集](https://github.com/wkgcass/Style/releases)v1.1.1  
>如果需要debug输出，则还依赖于log4j 1.2.17  
>推荐与Spring配合使用

##框架思想
以**已有的逻辑关系**代替**复杂的配置**。

##设计思路
通常写java时会使用Spring框架进行IoC管理。而Spring是基于bean的，配置起来首先需要将类映射到bean，然后描述bean的依赖关系。

事实上，很多时候依赖关系仅仅需要类型即可确定。比如说

	class A{
		...
		public void setB(B b){ this.b=b; }
	}
	
	class B{
	}

很显然，A依赖于B，A需要将一个B的实例通过setB注入进来。这应当是很自然的逻辑关系。

类型依赖在编码时就会设定好，所以，如果有工具能帮助完成这类“显而易见”的依赖该多好。于是我开发了Pure.IoC

#框架的框架
##扩展性
设计Pure.IoC时经历了不少思考。考虑下面这样的“复杂”情况（其实已经算不复杂了）

	@Singleton
	@Wire
	class Complex{
		private ......
		
		public Complex(){ ... }
		@Default public Complex(AnotherClass obj){ ... }
		
		public void setA(A a){ ... }
		public void setB(B b){ ... }	
		public void setInterface(Interf1 interf){ ... }
		public void setInterface(@Use(clazz=Impl2.class)Interf2 interf){ ... }
	}
	
	@Default(clazz=Impl1.class) interface Interf1{ ... }
	
上述注解描述了：

* @Singleton Complex是一个单例
* @Wire 在通过AutoWire构建时需要注入
* @Default 指定默认的构造器（构造函数上），指定默认的实现类（接口上）
* @Use 指定使用的类

注解应当可以无限的附加，也不能增加系统复杂性。  
所以，最终使用这种设计：

	AnnotationHandler + HandlerChain
AnnotationHandler分为四种，

* SetterAnnotationHandler 负责调用Setter，考虑setter，对应成员，和setter参数上的注解
* ParamAnnotationHandler 负责根据类型获取实例，考虑参数上的注解
* ConstructorFilter 负责选择构造器，考虑构造器上的注解
* TypeAnnotationHandler 负责选取要构造的类型，考虑类型上的注解

Handler被注册到IOCController上，根据注册的顺序进行handling操作。handler的内部实现有点像AOP，一般来说会调用chain.next().handle(...) ，然后根据返回值或者异常进行一些逻辑判断。详情见各Handler接口的handle方法文档

所以，可以非常方便的进行注解的扩展。

##循环依赖
循环依赖虽然不很常见，但没准会遇到。A->B->C->A  
从逻辑上，A,B,C中必须有一个是单例，否则对象的创建将无限循环下去。

另外：
Spring要求构造器不能循环依赖，否则无法完成构建。

不过Pure.IoC巧妙的解决了循环依赖的问题。

* 所有注入在构造时进行。只有注入完成，构造才会完成。
* 对于单例，在构造实际进行前把自己的引用交给IOCController，指示单例已经存在
* 由于只有注入完成才构造完成，所以不必担心获取到一个构造一半的对象。

所以，即使是构造器中包含循环依赖，也能顺利的进行（本质上所有依赖都是构造器依赖）。

##适用性
Pure.IoC由于设计为构造时注入，所以不需要任何额外组建即可放入框架中使用。也可配合Spring使用。甚至可以一部分setter通过本框架注入，一部分通过Spring注入。

直接与Struts2等相接也可以。由于注入过程完全在类内部，所以并不需要像Spring一样配置接管Struts2的对象工厂，而是直接就可使用（一个对象本身就是注入“工厂”）。

推荐的使用方法是

	extends AutoWire
	
但是有些情况下必须继承别的类，那么有两种解决办法：

1.

	@Wire class A { ... }
	
@Wire注解标注的类在任何经过Pure.IoC的情况下都会进行注入操作。  
在单独获取时可使用
	
	AutoWire.get(A.class)

获取实例

2.

	class A {
		public A(){
			AutoWire.wire(this);
		}
	}
	
构造时将自己的引用交给框架进行注入。

>实际上所有入口最终都调用AutoWire.wire(Object)

#如何使用？
首先使用上述三种方法任何一种进行框架接入。

##默认行为
没有任何注解参与时的默认行为是：

注入setter时，获取参数类型并向IOCController请求实例，获取实例后进行setter的invoke。

在获取实例时，取得唯一一个构造器，或者取得无参数的构造器。  
如果构造器有参数，则取得参数类型对应实例。  
最后进行构造

##扩展注解
###Setter

* Use(clazz, constant, variable)  
	**clazz**代表使用指定的类型作注入 接下来会对指定的类型进行Type检查。  
	**constant**代表之前在IOCController上注册的“常量”  
	**variable**代表之前在IOCController上注册的“变量”，必须是一些static的成员或者方法。

* Force(value)  
	将String类型的value转化为对应的类型。Force只支持基本类型和String
* Ignore  
	忽略该setter

###Type

* Wire
	指示该类需要注入
* IsSingleton  
	指示该类为单例
* Default(clazz)  
	一般用于interface或者abstract class  
	在Type检查时若遇到该注解则会转而构造该注解指定的类型

###Constructor

* Default()  
	在可能产生歧义时指定构造时默认使用的构造函数
	
###Param

* Use(clazz, constant, variable)  
	**clazz**代表使用指定的类型作注入 接下来会对指定的类型进行Type检查。  
	**constant**代表之前在IOCController上注册的“常量”  
	**variable**代表之前在IOCController上注册的“变量”，必须是一些static的成员或者方法。
* Force(value)  
	将String类型的value转化为对应的类型。Force只支持基本类型和String
	
>此外，进行Param处理时还提供自动的基本类型注入。  
>若没有找到可用的注入，则将检查基本类型和数组类型，并初始化为默认初始值（0，false，array[length=0]）