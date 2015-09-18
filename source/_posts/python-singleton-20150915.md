title: "在Python中实现单例模式"
date: 2015-09-15 15:17:58
tags: [Python, 技术]
categories: 技术 
---
有些时候你的项目中难免需要一些全局唯一的对象,这些对象大多是一些工具性的东西,在Python中实现单例模式并不是什么难事。以下总结几种方法:

##使用类装饰器
使用装饰器实现单例类的时候,类本身并不知道自己是单例的,所以写代码的人可以不care这个,只要正常写自己的类的实现就可以,类的单例有装饰器保证。

    def singleton(cls):
        instances = {}
        def _wrapper(*args, **kwargs):
            if cls not in instances:
                instances[cls] = cls(*args, **kwargs)
            return instances[cls]
        return _wrapper

##使用元类(`__metaclass__`)和可调用对象(`__call__`)
Python的对象系统中一些皆对象,类也不例外,可以称之为"类型对象",比较绕,但仔细思考也不难:类本身也是一种对象,只不过这种对象很特殊,它表示某一种类型。是对象,那必然是实例化来的,那么谁实例化后是这种类型对象呢?也就是元类。

Python中,`class`关键字表示定义一个类对象,此时解释器会按一定规则寻找`__metaclass__`,如果找到了,就调用对应的元类实现来实例化该类对象;没找到,就会调用`type`元类来实例化该类对象。

`__call__`是Python的魔术方法,Python的面向对象是"Duck type"的,意味着对象的行为可以通过实现协议来实现,可以看作是一种特殊的接口形式。某个类实现了`__call__`方法意味着该类的对象是可调用的,可以想像函数调用的样子。再考虑一下`foo=Foo()`这种实例化的形式,是不是很像啊。结合元类的概念,可以看出,`Foo`类是单例的,则在调用`Foo()`的时候每次都返回了同样的对象。而`Foo`作为一个类对象是单例的,意味着它的类(即生成它的元类)是实现了`__call__`方法的。所以可以如下实现:

    class Singleton(type):
        def __init__(cls, name, bases, attrs):
            super(Singleton, cls).__init__(name, bases, attrs)
            cls._instance = None
        def __call__(cls, *args, **kwargs):
            if cls._instance == None
                # 以下不要使用'cls._instance = cls(*args, **kwargs)', 防止死循环,
                # cls的调用行为已经被当前'__call__'协议拦截了
                # 使用super(Singleton, cls).__call__来生成cls的实例
                cls._instance = super(Singleton, cls).__call__(*args, **kwargs)
            return cls._instance

    class Foo(object): #单例类
        __metaclass__ = Singleton
    
    >>>a = Foo()
    >>>b = Foo()
    >>>a is b
    >>>True
    >>>a.x = 1
    >>>b.x
    >>>1

##使用`__new__`
`__init__`不是Python对象的构造方法,`__init__`只负责初始化实例对象,在调用`__init__`方法之前,会首先调用`__new__`方法生成对象,可以认为`__new__`方法充当了构造方法的角色。所以可以在`__new__`中加以控制,使得某个类只生成唯一对象。具体实现时可以实现一个父类,重载`__new__`方法,单例类只需要继承这个父类就好。

    class Singleton(object):
        def __new__(cls, *args, **kwargs):
            if not hasattr(cls, '_instance'):
                cls._instance = super(Singleton, cls).__new__(cls, *args, **kwargs)
            return cls._instance
    
    class Foo(Singleton): #单例类
        a = 1



