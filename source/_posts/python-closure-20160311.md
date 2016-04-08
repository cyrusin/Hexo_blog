title: "Python闭包的两个注意事项"
date: 2016-03-11 16:20:30
tags: [Python, 技术]
categories: Python
---

##延迟绑定

Python闭包函数所引用的外部自由变量是延迟绑定的。

    In [2]: def multipliers():
       ...:     return [lambda x: i * x for i in range(4)] 
    In [3]: print [m(2) for m in multipliers()]
    [6, 6, 6, 6]

如以上代码: `i`是闭包函数引用的外部作用域的自由变量, 只有在内部函数被调用的时候才会搜索变量`i`的值, 由于循环已结束, `i`指向最终值3, 所以各函数调用都得到了相同的结果。

解决方法:

1) 生成闭包函数的时候立即绑定(**使用函数形参的默认值**):

    In [5]: def multipliers():
        return [lambda x, i=i: i* x for i in range(4)]
           ...: 

    In [6]: print [m(2) for m in multipliers()]
    [0, 2, 4, 6]

如以上代码: 生成闭包函数的时候, 可以看到每个闭包函数都有一个带默认值的参数: `i=i`, 此时, 解释器会查找`i`的值, 并将其赋予形参`i`, 这样在生成闭包函数的外部作用域(即外部循环中), 找到了变量`i`, 遂将其当前值赋予形参`i`。

2) 使用**functools.partial**:

    In [26]: def multipliers():
        return [functools.partial(lambda i, x: x * i, i) for i in range(4)]
        ....: 

    In [27]: print [m(2) for m in multipliers()]
        [0, 2, 4, 6]

如以上代码: 在有可能因为延迟绑定而出问题的时候, 可以通过`functools.partial`构造偏函数, 使得自由变量优先绑定到闭包函数上。

##禁止在闭包函数内对引用的自由变量进行重新绑定

    def foo(func):
        free_value = 8
        def _wrapper(*args, **kwargs):
            old_free_value = free_value #保存旧的free_value
            free_value = old_free_value * 2 #模拟产生新的free_value
            func(*args, **kwargs)
            free_value = old_free_value
        return _wrapper

以上代码会报错, `UnboundLocalError: local variable 'free_value' referenced before assignment`, 以上代码本意是打算实现一个带有某个初始化状态(`free_value`)但在执行内部闭包函数的时候又可以按需变化出新的状态(`free_value = old_free_value * 2`)的装饰器, 但内部由于发生了重新绑定, 解释器会将`free_value`看作局部变量, `old_free_value = free_value`则会报错, 因为解释器认为`free_value`是没有赋值就被引用了。

解决：打算修改闭包函数引用的自由变量时, 可以将其放入一个list, 这样, `free_value = [8]`, `free_value`不可修改, 但`free_value[0]`是可以安全的被修改的。

另外, Python 3.x增加了`nonlocal`关键字, 也可以解决这个问题。
