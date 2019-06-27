---
title: python-metaclass
date: 2019-06-05 10:45:26
tags:
---
https://realpython.com/python-metaclasses/
https://docs.python.org/3/library/functions.html#type
https://www.cnblogs.com/tkqasn/p/6524879.html

```bash
➜  ~ python3
Python 3.7.3 (default, Mar 27 2019, 09:23:15)
[Clang 10.0.1 (clang-1001.0.46.3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>> Test = type('Test',(),{'bar':True})
>>> Test
<class '__main__.Test'>
>>> Test.bar
True
>>> def echo_bar(self):
...     print(self.bar)
...
>>> FooChild = type('FooChild',(Test,),{'echo_bar': echo_bar})
>>> aaa = FooChild()
>>> aaa.echo_bar()
True
>>> hasattr(FooChild, 'bar')
True
>>> hasattr(FooChild, 'echo_bar')
True
>>> FooChild.__class__
<class 'type'>
>>> FooChild.__class__.__class__
<class 'type'>
>>> bbb = FooChild()
>>> bbb.__class__
<class '__main__.FooChild'>
>>> bbb.__class__.__class__
<class 'type'>
>>>
```

```bash
>>>
>>> def my_meta_class(future_class_name, future_class_parents, future_class_attr):
...     attrs = ((name, value) for name, value in future_class_attr.items() if not name.startswith('__'))
...     uppercase_attr = dict((name.upper(), value) for name, value in attrs)
...     return type(future_class_name, future_class_parents, uppercase_attr)
...
>>>
>>> class FOO(object):
...     __metaclass__ = my_meta_class
...     bar = 'bar'
...
>>>
>>> dir(FOO)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__metaclass__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'bar']
>>>
>>>
>>> foo = FOO()
>>> dir(foo)
['__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__metaclass__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', 'bar']
>>>
>>>
```