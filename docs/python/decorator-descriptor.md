# Decorator descriptor

A simple use case of a decorator with __get__ method, turned into a non data-descriptor.

```python
class MyDeco:
    def __init__(self, func):
        self.func = func
 
    def __get__(self, instance, parent):
        func = functools.partial(self.__call__, instance)
        return functools.update_wrapper(func, self.func)
 
    def __call__(self, *args, **kwargs):
        result = self.func(*args, **kwargs)
        return result
```

usage:

```python
@MyDeco
def hello("name"):
    return f"Hello {name}"
 
class Person:
    @MyDeco
    def hello(self, name):
        return f"Hello {name}"
```