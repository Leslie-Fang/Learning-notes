---
title: Python类与继承
date: 2017-06-27 19:15:18
tags: 技术
---
## 类变量与实例变量
看这个[例子](!https://stackoverflow.com/questions/8701500/python-class-instance-variables-and-class-variables)
因为当用self.var去call一个变量的时候的顺序是**实例的dict->类的dict->基类**
也就是如果再实例里面找到这个变量相同的名字，就用实例的变量，没有找到就去找类变量，再没有找到就会找基类的变量，最后还没有找到就会报错
类变量还可以用 类名.变量名去调用
实例变量只可以用self.变量名

## 类的实例方法，静态方法与类方法
参考这个[链接](!https://www.zhihu.com/question/20021164)
1. 实例方法
形式：
class A(object):
  def function(self,var1):
最常见的方法包括了__init__等函数

2. 类方法
用@classmethod修饰，需要传入一个非self参数，这个参数的名字常写作cls或者cls_obj(也可以是其它名字，不是self), **可以被实例调用也可以被对象调用**
```
class Person(object):
    num = 0
    def __init__(self,name):
        self.name=name
        Person.num +=1
    @classmethod
    def get_nomber_of_instance(cls):
        return cls.num
if __name__ == "__main__":
    print("hello world")
    a = Person("bob")
    b = Person("bob2")
    print(Person.get_nomber_of_instance())
    print(a.get_nomber_of_instance())
```
这样的好处是在类的内部，通过第一个参数cls把类传递出来

<!--more-->
3. 静态方法
用@staticmethod修饰，它的适用场景决定了它一般不需要传入参数(self也不需要传入)
```
#needPrintName = "no" #yes or no
needPrintName = "yes"
class Person(object):
    num = 0
    def __init__(self,name):
        self.name=name
        Person.num +=1
    def printName(self):
        if self.needPrintName():
            print(self.name)
        else:
            print("Doesn't need to print name")
    @classmethod
    def get_nomber_of_instance(cls):
        return cls.num
    @staticmethod
    def needPrintName():
        return needPrintName=="yes"
if __name__ == "__main__":
    print("hello world")
    a = Person("bob")
    b = Person("bob2")
    print(Person.get_nomber_of_instance())
    print(a.get_nomber_of_instance())
    a.printName()
```
这样的好处是，有一些方法和类的功能相关，但是又不需要类或者对象本身去参与

## 类的继承
最基础的类就是object
class child_class(base_class):
注意点：
1. 派生类不会自动调用基类的__init__方法，需要在派生类的__init__函数里面去调用，base_class.__init__(self)
2. 调用基类的方法的时候，需要加上基类的类名前缀，**而且调用基类的函数记得带上self**
3. 总是先在本类中找方法，找不到再去基类里面找（派生类的方法会覆盖基类的方法）
```
class Person(object):
    num = 0
    def __init__(self,name):
        self.name=name
        Person.num +=1
    def printName(self):
        print(self.name)
    @classmethod
    def get_nomber_of_instance(cls):
        return cls.num
class Student(Person):
    def __init__(self,name,score):
        Person.__init__(self,name)
        self.score=score
if __name__ == "__main__":
    print("hello world")
    a = Person("bob")
    b = Person("bob2")
    print(Person.get_nomber_of_instance())
    print(a.get_nomber_of_instance())
    a.printName()
    b.printName()
    c = Student("bob3",100)
    c.printName()
```

**如果基类继承了object类，还可以用super去调用基类的方法**
推荐这种写法super(chrild_class,self)._init__(arg)(这里的arg不写self)，因为原来直接call基类的init函数存在这样的一种例外的[情况](!http://www.jackyshen.com/2015/08/19/multi-inheritance-with-super-in-Python/)
```
import sys
class Person(object):
    num = 0
    def __init__(self,name):
        self.name=name
        Person.num +=1
    def printName(self):
        print(self.name)
    @classmethod
    def get_nomber_of_instance(cls):
        return cls.num
class Student(Person):
    def __init__(self,name,score):
        super(self.__class__,self).__init__(name)
        self.score=score
if __name__ == "__main__":
    print('all argv:{}'.format(sys.argv))
    print(len(sys.argv))
    a = Person("bob")
    b = Person("bob2")
    print(Person.get_nomber_of_instance())
    print(a.get_nomber_of_instance())
    a.printName()
    b.printName()
    c = Student("bob3",100)
    c.printName()
```

## 备注
1. 获取对象的类 ins_obj.__class__.method(var)
```
def howperson(a):
    return a.__class__.num
class Person(object):
    num = 0
    def __init__(self,name):
        self.name=name
        Person.num +=1
if __name__ == "__main__":
    print("hello world")
    a = Person("bob")
    b = Person("bob2")
    print(howperson(a))
```
