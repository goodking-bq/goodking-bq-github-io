---
title: "回调函数"
date: 2020-05-14T15:46:37+08:00
draft: false
tags: ["python","reference"]
categories: ["python"]
---

## 什么是回调函数
>通过函数参数传递到其它代码的，某一块可执行代码的引用。这一设计允许了底层代码调用在高层定义的子程序。这种通过函数参数传的到函数内部再调用的就叫做回调函数

一个简单的例子
```
#a.py
def add_one(x):
    return x + 1

def add_two(x):
    return x + 2

def call(x, callback):
    return callback(x)

if __name__ == '__main__':
    print call(1, add_one)
    print call(1, add_two)
    
"""
#python a.py
2
3
"""
```
还有一个很经典的例子：
```
# b.py
class CallbackBase:
    def __init__(self):
        self.__callbackMap = {} # 注册函数对应
        for k in (getattr(self, x) for x in dir(self)):
            if hasattr(k, "bind_to_event"):
                self.__callbackMap.setdefault(k.bind_to_event, []).append(k)
            elif hasattr(k, "bind_to_event_list"):
                for j in k.bind_to_event_list:
                    self.__callbackMap.setdefault(j, []).append(k)
    @staticmethod
    def callback(event): #装饰器
        def f(g, ev = event):
            g.bind_to_event = ev
            return g
        return f

    @staticmethod
    def callbacklist(eventlist):
        def f(g, evl = eventlist):
            g.bind_to_event_list = evl
            return g
        return f

    def dispatch(self, event):
        l = self.__callbackMap[event] # l 为绑定的函数
        f = lambda *args, **kargs: \
            map(lambda x: x(*args, **kargs), l)
        return f


## Sample
class MyClass(CallbackBase):
    EVENT1 = 1
    EVENT2 = 2

    @CallbackBase.callback(EVENT1)
    def handler1(self, param = None): #绑定了一个事件
        print "handler1 with param: %s" % str(param)
        return None

    @CallbackBase.callbacklist([EVENT1, EVENT2])
    def handler2(self, param = None): # 绑定了 两个事件
        print "handler2 with param: %s" % str(param)
        return None

    def run(self, event, param = None):
        self.dispatch(event)(param)


if __name__ == "__main__":
    a = MyClass()
    a.run(MyClass.EVENT1, 'mandarina')
    a.run(MyClass.EVENT2, 'naranja')

"""
#python b.py
handler1 with param: mandarina
handler2 with param: mandarina
handler2 with param: naranja
"""
```
