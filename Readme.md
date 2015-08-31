点击[这里](http://blog.cassite.net/JAVA/Pure.IoC)或[这里](https://github.com/wkgcass/Pure.IoC/blob/master/Pure.IoCCN.md)获取中文文档。

#Pure.IoC
Pure.IoC is a light-weight type and annotation based dependency injection framework

>based on jdk 1.8  
>this framework uses [Style functional programming toolbox](https://github.com/wkgcass/Style/releases) v1.1.1  
>log4j 1.2.17 is required if you need debug output  
>It is recommended to use with Spring

##Philosophy
Replace **complex configuration** with **logical dependency**.

##Design idea
Usually we use *Spring* to do IoC management.However, Spring is based on *bean*s, It's required to map a class to a bean, then describe the dependencies of beans.

Infact, dependency only need to be determined by Type.  
e.g.

	class A{
		...
		public void setB(B b){ this.b=b; }
	}
	
	class B{
	}

It is clear that A depends on B. A require an instance of B to inject through setB. It's a natural logical relation.

Type dependency would be set when coding. So how great it would be if some tools/frameworks can help with this clear depending relation.  
As a result, i developed *Pure.IoC*

#Framework of framework
##Expansibility
It took me a lot time thiking how to design *Pure.IoC*.   
Take the following complex situation as an example (Actually it's still not very complex)

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
	
The annotations described:

* @Singleton Complex is a singleton
* @Wire Wire when goes through AutoWire
* @Default Choose the default constructor(the one on constructor), determine implement class(the one on interface)
* @Use determine which class to use

The annotations should be able to expand infinitely, and it should not cause the system to be more complex.  
Finally I designed the following pattern:

	AnnotationHandler + HandlerChain
AnnotationHandler is divided into 4 kinds:

* SetterAnnotationHandler Used to invoke setters, takes annotations on setters/correspoinding fields/parameters into consideration.
* ParamAnnotationHandler Used to retrieve instances with given type, takes annotations on parameters into consideration.
* ConstructorFilter Used to select constructors, takes annotations on constructors into consideration.
* TypeAnnotationHandler Used to choose a type, takes annotations on types.

Handler are reigstered into IOCController, and do handling operations according to the order when registering. The implement of handler looks like AOP, usually it will invoke chain.next().handle(...) , and do corresponding operations considering the result or exceptions from next handler.

See those handler's doc for more info.

As a result, it's very convenient to do expansion.

##Circular Dependencies
However it's not usual to occur circular dependencies，But we may have them sometimes. A->B->C->A  
At least one of A,B,C should be singleton, otherwise the construction would continue to go on.

Spring require constructors cannot have circular dependencies, otherwise the construction cannot finish.

However Pure.IoC solved the problem cleverly.

* All injection are performed in construction. The construction won't finish before injecting finished.
* For singletons, they hand over their references to IOCController to let the system know the singleton is already exist.
* The construction will finish only when injecting finished, so don't worry that the instantiating may not full。

As a result, even constructors have cirular dependencies, the injections can go on smoothly(actuall, all dependencies are constructor dependencies in the framework).

##Flexibility
Pure.IoC is designed as injecting when constructing, so no other modules are required putting this into other frameworks. It's a good idea to use this framework with spring. Even inject some setters with Pure.IoC and inject others with Spring.

Also it doesn't require any configuration when using with Struts2. The injection are inside the class, so it doesn't need to change object factory of Struts2.(an object itself is a factory)

The recommended way of using this framework:

	extends AutoWire
	
But in some circumstances, the class has already extended another class, there're 2 backup ways:

1.

	@Wire class A { ... }
	
Any class with @Wire annotation would be injected when going through Pure.IoC.

You can use:
	
	AutoWire.get(A.class)

to get instances.

2.

	class A {
		public A(){
			AutoWire.wire(this);
		}
	}
	
Handle over it's reference to framework when constructing.

>Actuall all entrances would call AutoWire.wire(Object)

#How to use ?
First of all, use these 3 methods presented to enable the injections.

##Default behavior
The default behaviors with no annotation presented are:

When injecting setter, obtain parameter type and request an instance from IOCController. Then invoke the setter.

When trying to get instances, get the only constructor, **or** get the constructor with no parameters.  
If the constructor has parameters, get instances with corresponding types.  
Finally do constructing.

##Extended Annotations
###Setter

* Use(clazz, constant, variable)  
	**clazz** use designated type as the argument. Then, type check would be invoked.  
	**constant** represents constants registered in IOCController.  
	**variable** represents variables registered in IOCController. The 'variables' must be *static* fileds or methods.

* Force(value)  
	Transform String value into proper type. Force only support primitives and Strings
* Ignore  
	Ignore the setter

###Type

* Wire
	indicates the class requires dependency injection.
* IsSingleton  
	indicates the class is a singleton
* Default(clazz)  
	Usually used on **interface**s or **abstract class**es  
	The constructing target would be redirected when doing Type check.

###Constructor

* Default()  
	indicates the constructor to use when choosing constructor method is ambiguous
	
###Param

* Use(clazz, constant, variable)  
	**clazz** use designated type as the argument. Then, type check would be invoked.  
	**constant** represents constants registered in IOCController.  
	**variable** represents variables registered in IOCController. The 'variables' must be *static* fileds or methods.

* Force(value)  
	Transform String value into proper type. Force only support primitives and Strings
	
>Also, param provide primitive type injection.  
>If no injections are found, it will check primitives and array types, and inject with initial values (0，false，array[length=0])