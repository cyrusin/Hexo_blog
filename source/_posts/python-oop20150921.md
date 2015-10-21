title: "Python中的元类,类,对象"
date: 2015-09-21 15:25:28
tags: [Python, 技术]
categories: Python
---
Python的"类型-对象"体系实现的简洁而优雅,"元类"-"类"-"实例对象"自上而下,层次分明。
以下是关键的几个点:

> 1.一切(函数,类等)皆对象(`object`)
>
> 2.变量名只是对象的引用
>
> 3.`object`类是所有类(对象)的最终父类,但object不是任何类的子类,这是对1的解释
>
> 4.`type`既是`object`类的实例,也是`object`类的子类
>
> 5.`object`作为一个类对象,她的类也是`type`
>
> 6.`object`和`type`是鸡和蛋的关系
>
> 7.`type`很重要,在整个体系中,真正干活的就是`type`
>
> 8.元类就是类(一切皆对象)的类
>
> 9.1)`type`负责生成'类'(对象),是最终**元类**(调用:`type(class_name, class_parents, attr_dict)`),`type`类有`__call__`方法,所以她生成的对象(类)都是可调用的
>
> 9.2)`type`负责生成实例对象,是**对象工厂**(让人意外,但事实如此:某类C生成`c_instance=C()`的过程实质是`type.__call__(C,...)`,9.1已经解释了,`type`的对象(C)是可调用的)
>
> 10.虽然一切皆对象,但对象生来不同
>
> 11.类型对象(包括元类,类)可以被子类化,可以继续被实例化
>
> 12.普通实例对象(自定义类型的实例，内建类型的实例)不可以被子类化,不可以被实例化

Python中对象到底是怎么生成的呢?看代码:

    class MetaClass(type):
        def __new__(cls, cls_name, cls_parents, attr_dict):
            print "MetaClass.__new__: create class istance: ", cls_name
            return super(MetaClass, cls).__new__(cls, cls_name, cls_parents, attr_dict)
    
        def __init__(self, *args, **kwargs):
            print "MetaClass.__init__: initialize: ", self
            super(MetaClass, self).__init__(*args, **kwargs)

        def __call__(self, *args, **kwargs):
            print "MetaClass.__call__: create instance: ", self
            return super(MetaClass, self).__call__(*args, **kwargs)
    
    def cls_decorator(cls):
        print "cls_decorator: ", cls
        return cls

    @cls_decorator
    class MyClass(object):
        __metaclass__ = MetaClass
        def __new__(cls, *args, **kwargs): print "MyClass.__new__, create instance of ", cls
            return super(MyClass, cls).__new__(cls, *args, **kwargs)

        def __init__(self, x):
            print "MyClass.__init__, instance is ", self
            self.x = x

    foo = MyClass(1)

以上代码执行后的输出:

    MetaClass.__new__: create class istance:  MyClass
    MetaClass.__init__: initialize:  <class '__main__.MyClass'>
    cls_decorator:  <class '__main__.MyClass'>
    MetaClass.__call__: create instance:  <class '__main__.MyClass'>
    MyClass.__new__, create instance of  <class '__main__.MyClass'>
    MyClass.__init__, instance is  <__main__.MyClass object at 0x7fbbd0303ed0>

解释如下:

>1. Python首先看类声明,准备三个传递给元类的参数。这三个参数分别为类名(cls_name),父类元组(cls_parents)以及属性字典(attr_dict)
>
>2. Python会检查`__metaclass__`属性,如果设置了此属性,它将调用metaclass,传递三个参数,并且返回一个类;如果没设此属性,它将去当前类的父类中寻找,如果父类也没有,就去当前类的module里找,如果也没有,直接使用type来充当元类生成类
>
>3. 在这个例子中,MetaClass自身就是一个类,这就意味着生成MyClass这个类的过程和一般的生成对象的过程一致:`MetaClass.__new__`将首先被调用，输入四个参数，这将新建一个MetaClass类的实例。然后这个实例的`MetaClass.__init__`将被调用,调用结果是作为一个新的类对象返回。所以此时MyClass将被设置成这个类对象
>
>4. 接下来Python将查看所有装饰了此类的装饰器。在这个例子中,只有一个装饰器。Python将调用这个装饰器,将从元类哪里得到的类传递给它作为参数。然后这个类将被装饰器返回的对象所替代
>
>5. 装饰器返回的类类型与元类设置的相同
>
>6. 当类被调用创建一个新的对象实例时,因为类的类型是MetaClass，因此Python将会调用元类的`__call__`方法。在这个例子中,`MetaClass.__call__`只是简单的调用了`type.__call__`,目的是创建一个传递给它的类的对象实例
>
>7. 下一步`type.__call__`通过`MyClass.__new__`创建一个对象
>
>8. 最后`type.__call__`通过`MyClass.__new__`返回的结果运行`MyClass.__init__`
>
>9. 返回的对象已经准备完毕


以上就是Python创建一个实例对象的过程。

参考: http://www.cafepy.com/article/python_types_and_objects
