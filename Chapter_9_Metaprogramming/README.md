# CHAPTER 9 Metaprogramming

소프트웨어 개발의 가장 중요한 원칙 중 하나는 "반복하지 말 것(don't repeat yourself)"이다. 반복적인 코드를 작성해야 하는 문제에 직면할 때마다, 더 우아한 솔루션을 찾는 것이 중요하다.
Python에서는 이러한 문제가 "metaprogramming" 범주에서 해결된다. 간단히 말해, metaprogramming은 코드를 조작(수정, 생성, 래핑)하는 것을 주요 목표로 하는 함수와 클래스를 만드는
것이다. 주요 기능으로는 decorator, class decorator, metaclass가 있다. 이 챕터의 주요 목적은 다양한 metaprogramming 기법을 탐구하고 Python의 동작을 커스터마이즈하는
방법의 예제를 제공하는 것이다.

## 9.1. 함수 주위에 Wrapper 배치

함수를 추가 코드로 래핑해야 하는 경우, decorator 함수를 정의한다.

```python
import time
from functools import wraps

def timethis(func):
    '''
    Decorator that reports the execution time.
    '''
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(func.__name__, end-start)
        return result
    return wrapper
```

사용 예:

```python
>>> @timethis
... def countdown(n):
...     '''
...     Counts down
...     '''
...     while n > 0:
...         n -= 1
...
>>> countdown(100000)
countdown 0.008917808532714844
```

### Discussion

Decorator는 함수를 입력으로 받아 새로운 함수를 출력으로 반환하는 함수다. 다음과 같은 코드를 작성할 때:

```python
@timethis
def countdown(n):
    ...
```

이것은 다음과 같은 별도의 단계를 수행한 것과 같다:

```python
def countdown(n):
    ...
countdown = timethis(countdown)
```

`@staticmethod`, `@classmethod`, `@property`와 같은 내장 decorator도 같은 방식으로 작동한다.

Decorator 내부의 코드는 일반적으로 `*args`와 `**kwargs`를 사용하여 모든 인자를 받는 새로운 함수를 생성하는 것을 포함한다. 이 함수 내부에서 원래 입력 함수에 대한 호출을 배치하고 그 결과를
반환한다. 그러나 추가하고 싶은 추가 코드(예: 타이밍)도 배치한다.

Decorator는 일반적으로 래핑되는 함수의 호출 시그니처나 반환 값을 변경하지 않는다는 점을 강조하는 것이 중요하다. `*args`와 `**kwargs`의 사용은 모든 입력 인자를 받을 수 있도록 하기 위한
것이다.

솔루션에서 `@wraps(func)` decorator의 사용은 잊기 쉽지만 함수 메타데이터 보존과 관련된 중요한 기술적 세부 사항이다.

## 9.2. Decorator 작성 시 함수 메타데이터 보존

Decorator를 작성할 때, functools 라이브러리의 `@wraps` decorator를 하위 wrapper 함수에 항상 적용해야 한다.

```python
import time
from functools import wraps

def timethis(func):
    '''
    Decorator that reports the execution time.
    '''
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(func.__name__, end-start)
        return result
    return wrapper
```

사용 예:

```python
>>> @timethis
... def countdown(n:int):
...     '''
...     Counts down
...     '''
...     while n > 0:
...         n -= 1
...
>>> countdown(100000)
countdown 0.008917808532714844
>>> countdown.__name__
'countdown'
>>> countdown.__doc__
'\n\tCounts down\n\t'
>>> countdown.__annotations__
{'n': <class 'int'>}
```

### Discussion

Decorator 메타데이터를 복사하는 것은 decorator를 작성하는 중요한 부분이다. `@wraps`를 사용하지 않으면 decorated 함수가 모든 종류의 유용한 정보를 잃게 된다.

`@wraps` decorator의 중요한 기능은 래핑된 함수를 `__wrapped__` 속성에서 사용할 수 있게 한다는 것이다. 예를 들어:

```python
>>> countdown.__wrapped__(100000)
>>>
```

`__wrapped__` 속성의 존재는 decorated 함수가 래핑된 함수의 기본 시그니처를 제대로 노출하게 한다.

```python
>>> from inspect import signature
>>> print(signature(countdown))
(n:int)
```

## 9.3. Decorator 언래핑

Decorator가 함수에 적용되었지만, 이를 "취소"하여 원래 언래핑된 함수에 접근하고 싶은 경우가 있다.

Decorator가 `@wraps`를 사용하여 제대로 구현되었다고 가정하면(Recipe 9.2 참조), 일반적으로 `__wrapped__` 속성에 접근하여 원래 함수에 접근할 수 있다.

```python
>>> @somedecorator
>>> def add(x, y):
...     return x + y
...
>>> orig_add = add.__wrapped__
>>> orig_add(3, 4)
7
```

### Discussion

Decorator 뒤의 언래핑된 함수에 직접 접근하는 것은 디버깅, 내부 검사 및 함수와 관련된 다른 작업에 유용할 수 있다. 그러나 이 레시피는 decorator의 구현이 functools 모듈의 `@wraps`를
사용하여 메타데이터를 제대로 복사하거나 `__wrapped__` 속성을 직접 설정한 경우에만 작동한다.

여러 decorator가 함수에 적용된 경우, `__wrapped__`에 접근하는 동작은 현재 정의되지 않았으며 피하는 것이 좋다.

## 9.4. 인자를 받는 Decorator 정의

인자를 받는 decorator 함수를 작성하고 싶은 경우가 있다.

다음은 함수에 로깅을 추가하되, 사용자가 로깅 레벨과 기타 세부 사항을 인자로 지정할 수 있도록 하는 decorator의 예다:

```python
from functools import wraps
import logging

def logged(level, name=None, message=None):
    '''
    Add logging to a function. level is the logging
    level, name is the logger name, and message is the
    log message. If name and message aren't specified,
    they default to the function's module and name.
    '''
    def decorate(func):
        logname = name if name else func.__module__
        log = logging.getLogger(logname)
        logmsg = message if message else func.__name__
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            log.log(level, logmsg)
            return func(*args, **kwargs)
        return wrapper
    return decorate

# 사용 예
@logged(logging.DEBUG)
def add(x, y):
    return x + y

@logged(logging.CRITICAL, 'example')
def spam():
    print('Spam!')
```

### Discussion

인자를 받는 decorator를 작성하는 것은 관련된 호출 시퀀스 때문에 까다롭다. 다음과 같은 코드가 있는 경우:

```python
@decorator(x, y, z)
def func(a, b):
    pass
```

decoration 프로세스는 다음과 같이 평가된다:

```python
def func(a, b):
    pass
func = decorator(x, y, z)(func)
```

`decorator(x, y, z)`의 결과는 함수를 입력으로 받아 래핑하는 callable이어야 한다는 점을 주의 깊게 관찰하라.

## 9.5. 사용자 조정 가능한 속성을 가진 Decorator 정의

함수를 래핑하되, 런타임에 decorator의 동작을 제어하는 데 사용할 수 있는 사용자 조정 가능한 속성을 가진 decorator 함수를 작성하고 싶다.

```python
from functools import wraps, partial
import logging

# obj의 속성으로 함수를 첨부하는 유틸리티 decorator
def attach_wrapper(obj, func=None):
    if func is None:
        return partial(attach_wrapper, obj)
    setattr(obj, func.__name__, func)
    return func

def logged(level, name=None, message=None):
    def decorate(func):
        logname = name if name else func.__module__
        log = logging.getLogger(logname)
        logmsg = message if message else func.__name__
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            log.log(level, logmsg)
            return func(*args, **kwargs)
        
        # setter 함수 첨부
        @attach_wrapper(wrapper)
        def set_level(newlevel):
            nonlocal level
            level = newlevel
        
        @attach_wrapper(wrapper)
        def set_message(newmsg):
            nonlocal logmsg
            logmsg = newmsg
        
        return wrapper
    return decorate

# 사용 예
@logged(logging.DEBUG)
def add(x, y):
    return x + y
```

사용 예:

```python
>>> import logging
>>> logging.basicConfig(level=logging.DEBUG)
>>> add(2, 3)
DEBUG:__main__:add
5
>>> # 로그 메시지 변경
>>> add.set_message('Add called')
>>> add(2, 3)
DEBUG:__main__:Add called
5
>>> # 로그 레벨 변경
>>> add.set_level(logging.WARNING)
>>> add(2, 3)
WARNING:__main__:Add called
5
```

### Discussion

이 레시피의 핵심은 wrapper에 속성으로 첨부되는 accessor 함수(예: `set_message()` 및 `set_level()`)에 있다. 각 accessor는 `nonlocal` 할당을 사용하여 내부
매개변수를 조정할 수 있게 한다.

이 레시피의 놀라운 기능은 accessor 함수가 여러 레벨의 decoration을 통해 전파된다는 것이다(모든 decorator가 `@functools.wraps`를 사용하는 경우).

## 9.6. 선택적 인자를 받는 Decorator 정의

`@decorator`와 같이 인자 없이 사용하거나 `@decorator(x,y,z)`와 같이 선택적 인자와 함께 사용할 수 있는 단일 decorator를 작성하고 싶다.

```python
from functools import wraps, partial
import logging

def logged(func=None, *, level=logging.DEBUG, name=None, message=None):
    if func is None:
        return partial(logged, level=level, name=name, message=message)
    
    logname = name if name else func.__module__
    log = logging.getLogger(logname)
    logmsg = message if message else func.__name__
    
    @wraps(func)
    def wrapper(*args, **kwargs):
        log.log(level, logmsg)
        return func(*args, **kwargs)
    return wrapper

# 사용 예
@logged
def add(x, y):
    return x + y

@logged(level=logging.CRITICAL, name='example')
def spam():
    print('Spam!')
```

### Discussion

이 레시피가 다루는 문제는 프로그래밍 일관성의 문제다. decorator를 사용할 때 대부분의 프로그래머는 인자 없이 또는 인자와 함께 적용하는 데 익숙하다.

코드가 어떻게 작동하는지 이해하려면 decorator가 함수에 어떻게 적용되는지와 호출 규칙에 대한 확실한 이해가 필요하다. 간단한 decorator의 경우:

```python
@logged
def add(x, y):
    return x + y
```

호출 시퀀스는 다음과 같다:

```python
def add(x, y):
    return x + y
add = logged(add)
```

이 경우 래핑할 함수가 `logged`의 첫 번째 인자로 전달된다.

인자를 받는 decorator의 경우:

```python
@logged(level=logging.CRITICAL, name='example')
def spam():
    print('Spam!')
```

호출 시퀀스는 다음과 같다:

```python
def spam():
    print('Spam!')
spam = logged(level=logging.CRITICAL, name='example')(spam)
```

`logged()`의 초기 호출 시 래핑할 함수가 전달되지 않는다. 따라서 decorator에서 선택적이어야 한다.

## 9.7. Decorator를 사용한 함수 타입 검사 강제

함수 인자에 대한 타입 검사를 일종의 assertion이나 contract로 선택적으로 강제하고 싶다.

```python
from inspect import signature
from functools import wraps

def typeassert(*ty_args, **ty_kwargs):
    def decorate(func):
        # 최적화 모드에서는 타입 검사 비활성화
        if not __debug__:
            return func
        
        # 함수 인자 이름을 제공된 타입에 매핑
        sig = signature(func)
        bound_types = sig.bind_partial(*ty_args, **ty_kwargs).arguments
        
        @wraps(func)
        def wrapper(*args, **kwargs):
            bound_values = sig.bind(*args, **kwargs)
            # 제공된 인자에 대해 타입 assertion 강제
            for name, value in bound_values.arguments.items():
                if name in bound_types:
                    if not isinstance(value, bound_types[name]):
                        raise TypeError(
                            'Argument {} must be {}'.format(name, bound_types[name])
                        )
            return func(*args, **kwargs)
        return wrapper
    return decorate
```

사용 예:

```python
>>> @typeassert(int, int)
... def add(x, y):
...     return x + y
...
>>> add(2, 3)
5
>>> add(2, 'hello')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "contract.py", line 33, in wrapper
TypeError: Argument y must be <class 'int'>
```

이 decorator는 매우 유연하여 함수 인자의 전체 또는 일부에 대해 타입을 지정할 수 있다. 또한 타입은 위치 또는 키워드로 지정할 수 있다.

```python
>>> @typeassert(int, z=int)
... def spam(x, y, z=42):
...     print(x, y, z)
...
>>> spam(1, 2, 3)
1 2 3
>>> spam(1, 'hello', 3)
1 hello 3
>>> spam(1, 'hello', 'world')
Traceback (most recent call last):
TypeError: Argument z must be <class 'int'>
```

### Discussion

이 레시피는 여러 중요하고 유용한 개념을 소개하는 고급 decorator 예제다.

Decorator의 한 측면은 함수 정의 시점에 한 번만 적용된다는 것이다. 특정 경우에 decorator가 추가한 기능을 비활성화하고 싶을 수 있다. 이를 위해 decorator 함수가 함수를 언래핑된 상태로
반환하게 하면 된다.

이 decorator를 작성하는 까다로운 부분은 래핑되는 함수의 인자 시그니처를 검사하고 작업하는 것을 포함한다는 것이다. 여기서 선택할 도구는 `inspect.signature()` 함수다.

## 9.8. 클래스의 일부로 Decorator 정의

클래스 정의 내부에 decorator를 정의하고 이를 다른 함수나 메서드에 적용하고 싶다.

클래스 내부에 decorator를 정의하는 것은 간단하지만, decorator가 인스턴스 메서드로 적용될지 클래스 메서드로 적용될지를 먼저 결정해야 한다.

```python
from functools import wraps

class A:
    # 인스턴스 메서드로서의 decorator
    def decorator1(self, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print('Decorator 1')
            return func(*args, **kwargs)
        return wrapper
    
    # 클래스 메서드로서의 decorator
    @classmethod
    def decorator2(cls, func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print('Decorator 2')
            return func(*args, **kwargs)
        return wrapper
```

사용 예:

```python
# 인스턴스 메서드로
a = A()
@a.decorator1
def spam():
    pass

# 클래스 메서드로
@A.decorator2
def grok():
    pass
```

### Discussion

클래스에 decorator를 정의하는 것은 처음에는 이상해 보일 수 있지만, 표준 라이브러리에 이런 예제가 있다. 특히 내장 `@property` decorator는 실제로 각각 decorator로 작동하는
`getter()`, `setter()`, `deleter()` 메서드를 가진 클래스다.

## 9.9. Decorator를 클래스로 정의

함수를 decorator로 래핑하고 싶지만, 결과가 callable 인스턴스가 될 것이다. decorator가 클래스 정의 내부와 외부 모두에서 작동하도록 해야 한다.

decorator를 인스턴스로 정의하려면 `__call__()` 및 `__get__()` 메서드를 구현해야 한다.

```python
import types
from functools import wraps

class Profiled:
    def __init__(self, func):
        wraps(func)(self)
        self.ncalls = 0
    
    def __call__(self, *args, **kwargs):
        self.ncalls += 1
        return self.__wrapped__(*args, **kwargs)
    
    def __get__(self, instance, cls):
        if instance is None:
            return self
        else:
            return types.MethodType(self, instance)
```

사용 예:

```python
@Profiled
def add(x, y):
    return x + y

class Spam:
    @Profiled
    def bar(self, x):
        print(self, x)
```

```python
>>> add(2, 3)
5
>>> add(4, 5)
9
>>> add.ncalls
2
>>> s = Spam()
>>> s.bar(1)
<__main__.Spam object at 0x10069e9d0> 1
>>> s.bar(2)
<__main__.Spam object at 0x10069e9d0> 2
>>> Spam.bar.ncalls
3
```

### Discussion

Decorator를 클래스로 정의하는 것은 일반적으로 간단하다. 그러나 특히 decorator를 인스턴스 메서드에 적용할 계획이라면 더 많은 설명이 필요한 미묘한 세부 사항이 있다.

`__get__()` 메서드를 생략하고 다른 모든 코드를 동일하게 유지하면, decorated 인스턴스 메서드를 호출하려고 할 때 이상한 일이 발생한다.

이유는 메서드를 구현하는 함수가 클래스에서 조회될 때마다 descriptor protocol의 일부로 `__get__()` 메서드가 호출되기 때문이다. 이 경우 `__get__()`의 목적은 bound method
객체를 생성하는 것이다(궁극적으로 메서드에 self 인자를 제공한다).

## 9.10. 클래스 및 정적 메서드에 Decorator 적용

클래스 또는 정적 메서드에 decorator를 적용하고 싶다.

클래스 및 정적 메서드에 decorator를 적용하는 것은 간단하지만, decorator가 `@classmethod` 또는 `@staticmethod` 전에 적용되어야 한다.

```python
import time
from functools import wraps

def timethis(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        r = func(*args, **kwargs)
        end = time.time()
        print(end-start)
        return r
    return wrapper

class Spam:
    @timethis
    def instance_method(self, n):
        print(self, n)
        while n > 0:
            n -= 1
    
    @classmethod
    @timethis
    def class_method(cls, n):
        print(cls, n)
        while n > 0:
            n -= 1
    
    @staticmethod
    @timethis
    def static_method(n):
        print(n)
        while n > 0:
            n -= 1
```

### Discussion

decorator의 순서가 잘못되면 오류가 발생한다. `@classmethod`와 `@staticmethod`는 실제로 직접 호출 가능한 객체를 생성하지 않는다. 대신 Recipe 8.9에 설명된 대로 특수
descriptor 객체를 생성한다. 따라서 다른 decorator에서 함수처럼 사용하려고 하면 decorator가 충돌한다. 이러한 decorator가 decorator 목록에서 먼저 나타나도록 하면 문제가
해결된다.

## 9.11. 래핑된 함수에 인자 추가하는 Decorator 작성

래핑된 함수의 호출 시그니처에 추가 인자를 추가하는 decorator를 작성하고 싶다. 그러나 추가된 인자는 함수의 기존 호출 규칙을 방해해서는 안 된다.

추가 인자는 keyword-only 인자를 사용하여 호출 시그니처에 주입될 수 있다.

```python
from functools import wraps

def optional_debug(func):
    @wraps(func)
    def wrapper(*args, debug=False, **kwargs):
        if debug:
            print('Calling', func.__name__)
        return func(*args, **kwargs)
    return wrapper
```

사용 예:

```python
>>> @optional_debug
... def spam(a,b,c):
...     print(a,b,c)
...
>>> spam(1,2,3)
1 2 3
>>> spam(1,2,3, debug=True)
Calling spam
1 2 3
```

### Discussion

래핑된 함수의 시그니처에 인자를 추가하는 것은 decorator를 사용하는 가장 일반적인 예가 아니다. 그러나 특정 종류의 코드 복제 패턴을 피하는 데 유용한 기법일 수 있다.

이 레시피의 구현은 keyword-only 인자가 `*args` 및 `**kwargs` 매개변수도 받는 함수에 쉽게 추가할 수 있다는 사실에 의존한다. keyword-only 인자를 사용하면 특별한 경우로 선택되어
나머지 positional 및 keyword 인자만 사용하는 후속 호출에서 제거된다.

## 9.12. Decorator를 사용하여 클래스 정의 패치

상속이나 metaclass를 사용하지 않고 클래스 정의의 일부를 검사하거나 다시 작성하여 동작을 변경하고 싶다.

이것은 class decorator의 완벽한 사용 사례일 수 있다. 예를 들어, 로깅을 수행하기 위해 `__getattribute__` 특수 메서드를 다시 작성하는 class decorator:

```python
def log_getattribute(cls):
    # 원래 구현 가져오기
    orig_getattribute = cls.__getattribute__
    
    # 새로운 정의 만들기
    def new_getattribute(self, name):
        print('getting:', name)
        return orig_getattribute(self, name)
    
    # 클래스에 첨부하고 반환
    cls.__getattribute__ = new_getattribute
    return cls

# 사용 예
@log_getattribute
class A:
    def __init__(self,x):
        self.x = x
    def spam(self):
        pass
```

사용:

```python
>>> a = A(42)
>>> a.x
getting: x
42
>>> a.spam()
getting: spam
```

### Discussion

Class decorator는 종종 mixin이나 metaclass와 관련된 다른 고급 기법에 대한 간단한 대안으로 사용될 수 있다.

## 9.13. Metaclass를 사용하여 인스턴스 생성 제어

singleton, 캐싱 또는 기타 유사한 기능을 구현하기 위해 인스턴스가 생성되는 방식을 변경하고 싶다.

Python 프로그래머가 알다시피, 클래스를 정의하면 함수처럼 호출하여 인스턴스를 생성한다. 이 단계를 커스터마이즈하려면 metaclass를 정의하고 `__call__()` 메서드를 어떤 방식으로든 재구현하면 된다.

아무도 인스턴스를 생성하지 못하게 하려면:

```python
class NoInstances(type):
    def __call__(self, *args, **kwargs):
        raise TypeError("Can't instantiate directly")

# 예
class Spam(metaclass=NoInstances):
    @staticmethod
    def grok(x):
        print('Spam.grok')
```

```python
>>> Spam.grok(42)
Spam.grok
>>> s = Spam()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "example1.py", line 7, in __call__
    raise TypeError("Can't instantiate directly")
TypeError: Can't instantiate directly
```

singleton 패턴을 구현하려면:

```python
class Singleton(type):
    def __init__(self, *args, **kwargs):
        self.__instance = None
        super().__init__(*args, **kwargs)
    
    def __call__(self, *args, **kwargs):
        if self.__instance is None:
            self.__instance = super().__call__(*args, **kwargs)
            return self.__instance
        else:
            return self.__instance

# 예
class Spam(metaclass=Singleton):
    def __init__(self):
        print('Creating Spam')
```

```python
>>> a = Spam()
Creating Spam
>>> b = Spam()
>>> a is b
True
>>> c = Spam()
>>> a is c
True
```

캐시된 인스턴스를 생성하려면:

```python
import weakref

class Cached(type):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.__cache = weakref.WeakValueDictionary()
    
    def __call__(self, *args):
        if args in self.__cache:
            return self.__cache[args]
        else:
            obj = super().__call__(*args)
            self.__cache[args] = obj
            return obj

# 예
class Spam(metaclass=Cached):
    def __init__(self, name):
        print('Creating Spam({!r})'.format(name))
        self.name = name
```

### Discussion

Metaclass를 사용하여 다양한 인스턴스 생성 패턴을 구현하는 것은 metaclass를 포함하지 않는 다른 솔루션보다 훨씬 더 우아한 접근 방식일 수 있다.

## 9.14. 클래스 속성 정의 순서 캡처

클래스 본문 내에서 속성과 메서드가 정의되는 순서를 자동으로 기록하여 다양한 작업(예: 직렬화, 데이터베이스 매핑 등)에 사용하고 싶다.

클래스 정의 본문에 대한 정보 캡처는 metaclass를 사용하여 쉽게 수행된다. 다음은 OrderedDict를 사용하여 descriptor의 정의 순서를 캡처하는 metaclass의 예다:

```python
from collections import OrderedDict

# 다양한 타입에 대한 descriptor 집합
class Typed:
    _expected_type = type(None)
    def __init__(self, name=None):
        self._name = name
    
    def __set__(self, instance, value):
        if not isinstance(value, self._expected_type):
            raise TypeError('Expected ' + str(self._expected_type))
        instance.__dict__[self._name] = value

class Integer(Typed):
    _expected_type = int

class Float(Typed):
    _expected_type = float

class String(Typed):
    _expected_type = str

# 클래스 본문에 OrderedDict를 사용하는 Metaclass
class OrderedMeta(type):
    def __new__(cls, clsname, bases, clsdict):
        d = dict(clsdict)
        order = []
        for name, value in clsdict.items():
            if isinstance(value, Typed):
                value._name = name
                order.append(name)
        d['_order'] = order
        return type.__new__(cls, clsname, bases, d)
    
    @classmethod
    def __prepare__(cls, clsname, bases):
        return OrderedDict()
```

사용 예:

```python
class Structure(metaclass=OrderedMeta):
    def as_csv(self):
        return ','.join(str(getattr(self,name)) for name in self._order)

class Stock(Structure):
    name = String()
    shares = Integer()
    price = Float()
    
    def __init__(self, name, shares, price):
        self.name = name
        self.shares = shares
        self.price = price
```

```python
>>> s = Stock('GOOG',100,490.1)
>>> s.name
'GOOG'
>>> s.as_csv()
'GOOG,100,490.1'
```

### Discussion

이 레시피의 전체 핵심은 OrderedMeta metaclass에 정의된 `__prepare__()` 메서드다. 이 메서드는 클래스 이름과 기본 클래스와 함께 클래스 정의 시작 시 즉시 호출된다. 그런 다음 클래스
본문을 처리할 때 사용할 매핑 객체를 반환해야 한다. 일반 딕셔너리 대신 OrderedDict를 반환하면 결과 정의 순서를 쉽게 캡처할 수 있다.

## 9.15. 선택적 인자를 받는 Metaclass 정의

타입 생성 중 처리의 측면을 제어하거나 구성하기 위해 클래스 정의가 선택적 인자를 제공할 수 있도록 하는 metaclass를 정의하고 싶다.

클래스를 정의할 때 Python은 class 문에서 metaclass 키워드 인자를 사용하여 metaclass를 지정할 수 있다. 그러나 사용자 정의 metaclass에서는 다음과 같이 추가 키워드 인자를 제공할 수
있다:

```python
class Spam(metaclass=MyMeta, debug=True, synchronize=True):
    ...
```

Metaclass에서 이러한 키워드 인자를 지원하려면 다음과 같이 keyword-only 인자를 사용하여 `__prepare__()`, `__new__()`, `__init__()` 메서드에 정의해야 한다:

```python
class MyMeta(type):
    # 선택적
    @classmethod
    def __prepare__(cls, name, bases, *, debug=False, synchronize=False):
        # 사용자 정의 처리
        ...
        return super().__prepare__(name, bases)
    
    # 필수
    def __new__(cls, name, bases, ns, *, debug=False, synchronize=False):
        # 사용자 정의 처리
        ...
        return super().__new__(cls, name, bases, ns)
    
    # 필수
    def __init__(self, name, bases, ns, *, debug=False, synchronize=False):
        # 사용자 정의 처리
        ...
        super().__init__(name, bases, ns)
```

### Discussion

Metaclass에 선택적 키워드 인자를 추가하려면 클래스 생성과 관련된 모든 단계를 이해해야 한다. `__prepare__()` 메서드가 먼저 호출되며 클래스 본문이 처리되기 전에 클래스 네임스페이스를 생성하는 데
사용된다. `__new__()` 메서드는 결과 타입 객체를 인스턴스화하는 데 사용된다. `__init__()` 메서드는 마지막으로 호출되며 추가 초기화 단계를 수행하는 데 사용된다.

## 9.16. *args 및 **kwargs에 대한 인자 시그니처 강제

`*args` 및 `**kwargs`를 사용하는 함수나 메서드를 작성했지만, 전달된 인자가 특정 함수 호출 시그니처와 일치하는지 확인하고 싶다.

함수 호출 시그니처를 조작하려는 모든 문제에 대해 inspect 모듈에 있는 signature 기능을 사용해야 한다. 두 클래스 `Signature`와 `Parameter`가 특히 중요하다.

```python
>>> from inspect import Signature, Parameter
>>> # func(x, y=42, *, z=None)에 대한 시그니처 만들기
>>> parms = [ Parameter('x', Parameter.POSITIONAL_OR_KEYWORD),
...           Parameter('y', Parameter.POSITIONAL_OR_KEYWORD, default=42),
...           Parameter('z', Parameter.KEYWORD_ONLY, default=None) ]
>>> sig = Signature(parms)
>>> print(sig)
(x, y=42, *, z=None)
```

signature 객체가 있으면 signature의 `bind()` 메서드를 사용하여 쉽게 `*args` 및 `**kwargs`에 바인딩할 수 있다:

```python
>>> def func(*args, **kwargs):
...     bound_values = sig.bind(*args, **kwargs)
...     for name, value in bound_values.arguments.items():
...         print(name,value)
...
>>> func(1, 2, z=3)
x 1
y 2
z 3
>>> func(1)
x 1
>>> func(1, z=3)
x 1
z 3
```

더 구체적인 예로, 기본 클래스가 매우 범용적인 버전의 `__init__()`을 정의했지만 서브클래스가 예상 시그니처를 제공해야 하는 경우:

```python
from inspect import Signature, Parameter

def make_sig(*names):
    parms = [Parameter(name, Parameter.POSITIONAL_OR_KEYWORD)
             for name in names]
    return Signature(parms)

class Structure:
    __signature__ = make_sig()
    def __init__(self, *args, **kwargs):
        bound_values = self.__signature__.bind(*args, **kwargs)
        for name, value in bound_values.arguments.items():
            setattr(self, name, value)

# 사용 예
class Stock(Structure):
    __signature__ = make_sig('name', 'shares', 'price')

class Point(Structure):
    __signature__ = make_sig('x', 'y')
```

```python
>>> import inspect
>>> print(inspect.signature(Stock))
(name, shares, price)
>>> s1 = Stock('ACME', 100, 490.1)
>>> s2 = Stock('ACME', 100)
Traceback (most recent call last):
...
TypeError: 'price' parameter lacking default value
```

### Discussion

`*args` 및 `**kwargs`를 포함하는 함수를 사용하는 것은 범용 라이브러리를 만들거나 decorator를 작성하거나 proxy를 구현할 때 매우 일반적이다. 그러나 이러한 함수의 한 가지 단점은 자체 인자
검사를 구현하려는 경우 빠르게 다루기 힘든 혼란이 될 수 있다는 것이다. signature 객체의 사용은 이를 단순화한다.

## 9.17. 클래스에서 코딩 규칙 강제

프로그램이 대규모 클래스 계층 구조로 구성되어 있으며 프로그래머의 정신을 유지하는 데 도움이 되도록 특정 종류의 코딩 규칙을 강제하고 싶다.

클래스의 정의를 모니터링하려면 종종 metaclass를 정의하여 수행할 수 있다. 기본 metaclass는 일반적으로 type에서 상속하고 `__new__()` 메서드나 `__init__()` 메서드를 재정의하여
정의된다.

```python
class MyMeta(type):
    def __new__(self, clsname, bases, clsdict):
        # clsname은 정의되는 클래스의 이름
        # bases는 기본 클래스의 튜플
        # clsdict는 클래스 딕셔너리
        return super().__new__(cls, clsname, bases, clsdict)
```

Metaclass를 사용하려면 일반적으로 다른 객체가 상속하는 최상위 기본 클래스에 통합한다:

```python
class Root(metaclass=MyMeta):
    pass

class A(Root):
    pass

class B(Root):
    pass
```

Metaclass의 주요 기능은 정의 시점에 클래스의 내용을 검사할 수 있다는 것이다. 재정의된 `__init__()` 메서드 내부에서 클래스 딕셔너리, 기본 클래스 등을 자유롭게 검사할 수 있다. 또한
metaclass가 클래스에 대해 지정되면 모든 서브클래스에 의해 상속된다.

구체적이지만 기발한 예로, 혼합 대소문자 이름을 가진 메서드를 포함하는 클래스 정의를 거부하는 metaclass:

```python
class NoMixedCaseMeta(type):
    def __new__(cls, clsname, bases, clsdict):
        for name in clsdict:
            if name.lower() != name:
                raise TypeError('Bad attribute name: ' + name)
        return super().__new__(cls, clsname, bases, clsdict)

class Root(metaclass=NoMixedCaseMeta):
    pass

class A(Root):
    def foo_bar(self):  # Ok
        pass

class B(Root):
    def fooBar(self):  # TypeError
        pass
```

더 고급스럽고 유용한 예로, 재정의된 메서드의 정의를 검사하여 슈퍼클래스의 원래 메서드와 동일한 호출 시그니처를 가지는지 확인하는 metaclass:

```python
from inspect import signature
import logging

class MatchSignaturesMeta(type):
    def __init__(self, clsname, bases, clsdict):
        super().__init__(clsname, bases, clsdict)
        sup = super(self, self)
        for name, value in clsdict.items():
            if name.startswith('_') or not callable(value):
                continue
            # 이전 정의 가져오기(있는 경우) 및 시그니처 비교
            prev_dfn = getattr(sup,name,None)
            if prev_dfn:
                prev_sig = signature(prev_dfn)
                val_sig = signature(value)
                if prev_sig != val_sig:
                    logging.warning('Signature mismatch in %s. %s != %s',
                                    value.__qualname__, prev_sig, val_sig)

# 예
class Root(metaclass=MatchSignaturesMeta):
    pass

class A(Root):
    def foo(self, x, y):
        pass
    
    def spam(self, x, *, z):
        pass

class B(A):
    def foo(self, a, b):
        pass
    
    def spam(self,x,z):
        pass
```

실행하면 다음과 같은 출력을 얻는다:

```
WARNING:root:Signature mismatch in B.spam. (self, x, *, z) != (self, x, z)
WARNING:root:Signature mismatch in B.foo. (self, x, y) != (self, a, b)
```

## 9.18. 프로그래밍 방식으로 클래스 정의

새로운 클래스 객체를 생성해야 하는 코드를 작성하고 있다. 클래스 소스 코드를 문자열로 emit하고 `exec()`와 같은 함수를 사용하여 평가하는 것을 고려했지만, 더 우아한 솔루션을 선호한다.

`types.new_class()` 함수를 사용하여 새로운 클래스 객체를 인스턴스화할 수 있다. 클래스의 이름, 부모 클래스의 튜플, 키워드 인자, 클래스 딕셔너리를 멤버로 채우는 콜백만 제공하면 된다.

```python
# 메서드
def __init__(self, name, shares, price):
    self.name = name
    self.shares = shares
    self.price = price

def cost(self):
    return self.shares * self.price

cls_dict = {
    '__init__' : __init__,
    'cost' : cost,
}

# 클래스 만들기
import types

Stock = types.new_class('Stock', (), {}, lambda ns: ns.update(cls_dict))
Stock.__module__ = __name__
```

이것은 예상대로 작동하는 일반 클래스 객체를 만든다:

```python
>>> s = Stock('ACME', 50, 91.1)
>>> s
<stock.Stock object at 0x1006a9b10>
>>> s.cost()
4555.0
```

### Discussion

새로운 클래스 객체를 제조할 수 있는 것은 특정 맥락에서 유용할 수 있다. 더 친숙한 예 중 하나는 `collections.namedtuple()` 함수다.

## 9.19. 정의 시점에 클래스 멤버 초기화

인스턴스가 생성될 때가 아니라 클래스가 정의될 때 클래스 정의의 일부를 한 번 초기화하고 싶다.

클래스 정의 시점에 초기화 또는 설정 작업을 수행하는 것은 metaclass의 고전적인 사용 사례다. 본질적으로 metaclass는 정의 시점에 트리거되며, 이 시점에서 추가 단계를 수행할 수 있다.

```python
import operator

class StructTupleMeta(type):
    def __init__(cls, *args, **kwargs):
        super().__init__(*args, **kwargs)
        for n, name in enumerate(cls._fields):
            setattr(cls, name, property(operator.itemgetter(n)))

class StructTuple(tuple, metaclass=StructTupleMeta):
    _fields = []
    def __new__(cls, *args):
        if len(args) != len(cls._fields):
            raise ValueError('{} arguments required'.format(len(cls._fields)))
        return super().__new__(cls,args)
```

사용 예:

```python
class Stock(StructTuple):
    _fields = ['name', 'shares', 'price']

class Point(StructTuple):
    _fields = ['x', 'y']
```

```python
>>> s = Stock('ACME', 50, 91.1)
>>> s
('ACME', 50, 91.1)
>>> s[0]
'ACME'
>>> s.name
'ACME'
>>> s.shares * s.price
4555.0
```

### Discussion

이 레시피에서 `StructTupleMeta` 클래스는 `_fields` 클래스 속성의 속성 이름 목록을 가져와 특정 튜플 슬롯에 접근하는 property 메서드로 전환한다.
`operator.itemgetter()` 함수는 accessor 함수를 생성하고 `property()` 함수는 이를 property로 전환한다.

이 레시피의 가장 까다로운 부분은 다른 초기화 단계가 언제 발생하는지 아는 것이다. `StructTupleMeta`의 `__init__()` 메서드는 정의된 각 클래스에 대해 한 번만 호출된다.

## 9.20. 함수 어노테이션을 사용한 Multiple Dispatch 구현

함수 인자 어노테이션에 대해 배웠고 이를 사용하여 타입을 기반으로 multiple-dispatch(메서드 오버로딩)를 구현할 수 있다고 생각했다.

이 레시피는 간단한 관찰을 기반으로 한다. Python이 인자에 어노테이션을 허용하기 때문에 다음과 같은 코드를 작성할 수 있을 것이다:

```python
class Spam:
    def bar(self, x:int, y:int):
        print('Bar 1:', x, y)
    
    def bar(self, s:str, n:int = 0):
        print('Bar 2:', s, n)

s = Spam()
s.bar(2, 3)      # Prints Bar 1: 2 3
s.bar('hello')   # Prints Bar 2: hello 0
```

다음은 metaclass와 descriptor의 조합을 사용하여 이를 수행하는 솔루션의 시작이다:

```python
import inspect
import types

class MultiMethod:
    '''
    단일 multimethod를 나타냄.
    '''
    def __init__(self, name):
        self._methods = {}
        self.__name__ = name
    
    def register(self, meth):
        '''
        multimethod로 새 메서드 등록
        '''
        sig = inspect.signature(meth)
        # 메서드의 어노테이션에서 타입 시그니처 구축
        types = []
        for name, parm in sig.parameters.items():
            if name == 'self':
                continue
            if parm.annotation is inspect.Parameter.empty:
                raise TypeError(
                    'Argument {} must be annotated with a type'.format(name)
                )
            if not isinstance(parm.annotation, type):
                raise TypeError(
                    'Argument {} annotation must be a type'.format(name)
                )
            if parm.default is not inspect.Parameter.empty:
                self._methods[tuple(types)] = meth
            types.append(parm.annotation)
        
        self._methods[tuple(types)] = meth
    
    def __call__(self, *args):
        '''
        인자의 타입 시그니처를 기반으로 메서드 호출
        '''
        types = tuple(type(arg) for arg in args[1:])
        meth = self._methods.get(types, None)
        if meth:
            return meth(*args)
        else:
            raise TypeError('No matching method for types {}'.format(types))
    
    def __get__(self, instance, cls):
        '''
        클래스에서 호출이 작동하도록 하는 데 필요한 Descriptor 메서드
        '''
        if instance is not None:
            return types.MethodType(self, instance)
        else:
            return self

class MultiDict(dict):
    '''
    metaclass에서 multimethod를 구축하기 위한 특수 딕셔너리
    '''
    def __setitem__(self, key, value):
        if key in self:
            current_value = self[key]
            if isinstance(current_value, MultiMethod):
                current_value.register(value)
            else:
                mvalue = MultiMethod(key)
                mvalue.register(current_value)
                mvalue.register(value)
                super().__setitem__(key, mvalue)
        else:
            super().__setitem__(key, value)

class MultipleMeta(type):
    '''
    메서드의 multiple dispatch를 허용하는 Metaclass
    '''
    def __new__(cls, clsname, bases, clsdict):
        return type.__new__(cls, clsname, bases, dict(clsdict))
    
    @classmethod
    def __prepare__(cls, clsname, bases):
        return MultiDict()
```

사용 예:

```python
class Spam(metaclass=MultipleMeta):
    def bar(self, x:int, y:int):
        print('Bar 1:', x, y)
    
    def bar(self, s:str, n:int = 0):
        print('Bar 2:', s, n)
```

```python
>>> s = Spam()
>>> s.bar(2, 3)
Bar 1: 2 3
>>> s.bar('hello')
Bar 2: hello 0
>>> s.bar('hello', 5)
Bar 2: hello 5
>>> s.bar(2, 'hello')
Traceback (most recent call last):
...
TypeError: No matching method for types (<class 'int'>, <class 'str'>)
```

### Discussion

구현의 주요 아이디어는 비교적 간단하다. `MutipleMeta` metaclass는 `__prepare__()` 메서드를 사용하여 `MultiDict`의 인스턴스로 사용자 정의 클래스 딕셔너리를 제공한다. 일반
딕셔너리와 달리 `MultiDict`는 항목이 설정될 때 항목이 이미 존재하는지 확인한다. 존재하는 경우 중복 항목이 `MultiMethod`의 인스턴스 내부에 병합된다.

`MultiMethod`의 인스턴스는 타입 시그니처에서 함수로의 매핑을 구축하여 메서드를 수집한다. 구성 중에 함수 어노테이션이 이러한 시그니처를 수집하고 매핑을 구축하는 데 사용된다.

## 9.21. 반복적인 Property 메서드 피하기

반복적으로 타입 검사와 같은 일반적인 작업을 수행하는 property 메서드를 정의하는 클래스를 작성하고 있다. 코드를 단순화하여 코드 반복이 많지 않도록 하고 싶다.

간단한 클래스를 고려해보자:

```python
class Person:
    def __init__(self, name ,age):
        self.name = name
        self.age = age
    
    @property
    def name(self):
        return self._name
    
    @name.setter
    def name(self, value):
        if not isinstance(value, str):
            raise TypeError('name must be a string')
        self._name = value
    
    @property
    def age(self):
        return self._age
    
    @age.setter
    def age(self, value):
        if not isinstance(value, int):
            raise TypeError('age must be an int')
        self._age = value
```

많은 코드가 속성 값에 대한 타입 assertion을 강제하기 위해 작성되고 있다. 이러한 코드를 볼 때마다 이를 단순화하는 다양한 방법을 탐색해야 한다. 한 가지 가능한 접근 방식은 property를 정의하고
반환하는 함수를 만드는 것이다:

```python
def typed_property(name, expected_type):
    storage_name = '_' + name
    
    @property
    def prop(self):
        return getattr(self, storage_name)
    
    @prop.setter
    def prop(self, value):
        if not isinstance(value, expected_type):
            raise TypeError('{} must be a {}'.format(name, expected_type))
        setattr(self, storage_name, value)
    
    return prop

# 사용 예
class Person:
    name = typed_property('name', str)
    age = typed_property('age', int)
    
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

### Discussion

이 레시피는 내부 함수나 클로저의 중요한 기능, 즉 매크로처럼 작동하는 코드를 작성하는 데 사용하는 것을 보여준다. `typed_property()` 함수는 이상하게 보일 수 있지만, 실제로는 property 코드를
생성하고 결과 property 객체를 반환하는 것일 뿐이다.

이 레시피는 `functools.partial()` 함수를 사용하여 흥미로운 방식으로 조정할 수 있다:

```python
from functools import partial

String = partial(typed_property, expected_type=str)
Integer = partial(typed_property, expected_type=int)

# 예:
class Person:
    name = String('name')
    age = Integer('age')
    
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

## 9.22. Context Manager를 쉽게 정의

`with` 문과 함께 사용할 새로운 종류의 context manager를 구현하고 싶다.

새로운 context manager를 작성하는 가장 간단한 방법 중 하나는 contextlib 모듈의 `@contextmanager` decorator를 사용하는 것이다.

```python
import time
from contextlib import contextmanager

@contextmanager
def timethis(label):
    start = time.time()
    try:
        yield
    finally:
        end = time.time()
        print('{}: {}'.format(label, end - start))

# 사용 예
with timethis('counting'):
    n = 10000000
    while n > 0:
        n -= 1
```

`timethis()` 함수에서 yield 이전의 모든 코드는 context manager의 `__enter__()` 메서드로 실행된다. yield 이후의 모든 코드는 `__exit__()` 메서드로 실행된다.
예외가 있으면 yield 문에서 발생한다.

리스트 객체에 대한 일종의 트랜잭션을 구현하는 약간 더 고급스러운 context manager:

```python
@contextmanager
def list_transaction(orig_list):
    working = list(orig_list)
    yield working
    orig_list[:] = working
```

아이디어는 전체 코드 블록이 예외 없이 완료될 때만 리스트에 대한 변경 사항이 적용된다는 것이다:

```python
>>> items = [1, 2, 3]
>>> with list_transaction(items) as working:
...     working.append(4)
...     working.append(5)
...
>>> items
[1, 2, 3, 4, 5]
>>> with list_transaction(items) as working:
...     working.append(6)
...     working.append(7)
...     raise RuntimeError('oops')
...
Traceback (most recent call last):
  File "<stdin>", line 4, in <module>
RuntimeError: oops
>>> items
[1, 2, 3, 4, 5]
```

### Discussion

일반적으로 context manager를 작성하려면 `__enter__()` 및 `__exit__()` 메서드를 가진 클래스를 정의한다. 이것은 어렵지 않지만 `@contextmanager`를 사용하여 간단한 함수를
작성하는 것보다 훨씬 더 지루하다.

`@contextmanager`는 실제로 자체 포함된 context-management 함수를 작성하는 데만 사용된다. with 문을 지원해야 하는 객체(예: 파일, 네트워크 연결 또는 lock)가 있는 경우 여전히
`__enter__()` 및 `__exit__()` 메서드를 별도로 구현해야 한다.

## 9.23. Local Side Effect를 가진 코드 실행

`exec()`를 사용하여 호출자의 범위에서 코드 조각을 실행하고 있지만, 실행 후 결과가 보이지 않는다.

문제를 더 잘 이해하기 위해 작은 실험을 해보자. 먼저 전역 네임스페이스에서 코드 조각을 실행한다:

```python
>>> a = 13
>>> exec('b = a + 1')
>>> print(b)
14
```

이제 함수 내부에서 같은 실험을 시도한다:

```python
>>> def test():
...     a = 13
...     exec('b = a + 1')
...     print(b)
...
>>> test()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 4, in test
NameError: global name 'b' is not defined
```

이러한 종류의 문제를 해결하려면 `exec()` 호출 전에 `locals()` 함수를 사용하여 로컬 변수의 딕셔너리를 얻어야 한다. 즉시 그 후에 locals 딕셔너리에서 수정된 값을 추출할 수 있다:

```python
>>> def test():
...     a = 13
...     loc = locals()
...     exec('b = a + 1')
...     b = loc['b']
...     print(b)
...
>>> test()
14
```

### Discussion

`exec()`의 올바른 사용은 실제로 꽤 까다롭다. 사실 `exec()`의 사용을 고려할 수 있는 대부분의 상황에서 더 우아한 솔루션이 존재할 가능성이 높다(예: decorator, 클로저, metaclass
등).

## 9.24. Python 소스 파싱 및 분석

Python 소스 코드를 파싱하고 분석하는 프로그램을 작성하고 싶다.

대부분의 프로그래머는 Python이 문자열 형태로 제공된 소스 코드를 평가하거나 실행할 수 있다는 것을 알고 있다:

```python
>>> x = 42
>>> eval('2 + 3*4 + x')
56
>>> exec('for i in range(10): print(i)')
0
1
2
...
9
```

그러나 ast 모듈을 사용하여 Python 소스 코드를 분석할 수 있는 추상 구문 트리(AST)로 컴파일할 수 있다:

```python
>>> import ast
>>> ex = ast.parse('2 + 3*4 + x', mode='eval')
>>> ex
<_ast.Expression object at 0x1007473d0>
>>> ast.dump(ex)
"Expression(body=BinOp(left=BinOp(left=Num(n=2), op=Add(),
right=BinOp(left=Num(n=3), op=Mult(), right=Num(n=4))), op=Add(),
right=Name(id='x', ctx=Load())))"
```

소스 트리를 분석하려면 약간의 학습이 필요하지만, AST 노드의 컬렉션으로 구성된다. 이러한 노드로 작업하는 가장 쉬운 방법은 관심 있는 노드와 일치하는 다양한 `visit_NodeName()` 메서드를 구현하는
visitor 클래스를 정의하는 것이다. 다음은 로드, 저장 및 삭제되는 이름에 대한 정보를 기록하는 클래스의 예다:

```python
import ast

class CodeAnalyzer(ast.NodeVisitor):
    def __init__(self):
        self.loaded = set()
        self.stored = set()
        self.deleted = set()
    
    def visit_Name(self, node):
        if isinstance(node.ctx, ast.Load):
            self.loaded.add(node.id)
        elif isinstance(node.ctx, ast.Store):
            self.stored.add(node.id)
        elif isinstance(node.ctx, ast.Del):
            self.deleted.add(node.id)

# 사용 예
if __name__ == '__main__':
    code = '''
for i in range(10):
    print(i)
del i
'''
    
    top = ast.parse(code, mode='exec')
    c = CodeAnalyzer()
    c.visit(top)
    print('Loaded:', c.loaded)
    print('Stored:', c.stored)
    print('Deleted:', c.deleted)
```

출력:

```
Loaded: {'i', 'range', 'print'}
Stored: {'i'}
Deleted: {'i'}
```

마지막으로 AST는 `compile()` 함수를 사용하여 컴파일되고 실행될 수 있다:

```python
>>> exec(compile(top,'<stdin>', 'exec'))
0
1
2
...
9
```

### Discussion

소스 코드를 분석하고 정보를 얻을 수 있다는 사실은 다양한 코드 분석, 최적화 또는 검증 도구를 작성하는 시작점이 될 수 있다.

## 9.25. Python Byte Code 디스어셈블

코드가 내부적으로 무엇을 하는지 세부적으로 알고 싶어서 인터프리터가 사용하는 하위 레벨 byte code로 디스어셈블하고 싶다.

dis 모듈을 사용하여 모든 Python 함수의 디스어셈블리를 출력할 수 있다:

```python
>>> def countdown(n):
...     while n > 0:
...         print('T-minus', n)
...         n -= 1
...     print('Blastoff!')
...
>>> import dis
>>> dis.dis(countdown)
  2           0 SETUP_LOOP              39 (to 42)
        >>    3 LOAD_FAST                0 (n)
              6 LOAD_CONST               1 (0)
              9 COMPARE_OP               4 (>)
             12 POP_JUMP_IF_FALSE       41
  3          15 LOAD_GLOBAL              0 (print)
             18 LOAD_CONST               2 ('T-minus')
             21 LOAD_FAST                0 (n)
             24 CALL_FUNCTION            2 (2 positional, 0 keyword pair)
             27 POP_TOP
  4          28 LOAD_FAST                0 (n)
             31 LOAD_CONST               3 (1)
             34 INPLACE_SUBTRACT
             35 STORE_FAST               0 (n)
             38 JUMP_ABSOLUTE            3
        >>   41 POP_BLOCK
  5     >>   42 LOAD_GLOBAL              0 (print)
             45 LOAD_CONST               4 ('Blastoff!')
             48 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
             51 POP_TOP
             52 LOAD_CONST               0 (None)
             55 RETURN_VALUE
```

### Discussion

dis 모듈은 프로그램에서 매우 낮은 수준에서 무슨 일이 일어나고 있는지 연구해야 하는 경우(예: 성능 특성을 이해하려는 경우) 유용할 수 있다.

`dis()` 함수에 의해 해석되는 원시 byte code는 다음과 같이 함수에서 사용할 수 있다:

```python
>>> countdown.__code__.co_code
b"x'\x00|\x00\x00d\x01\x00k\x04\x00r)\x00t\x00\x00d\x02\x00|\x00\x00\x83
\x02\x00\x01|\x00\x00d\x03\x008}\x00\x00q\x03\x00Wt\x00\x00d\x04\x00\x83
\x01\x00\x01d\x00\x00S"
```

이 코드를 직접 해석하려면 opcode 모듈에 정의된 일부 상수를 사용해야 한다.