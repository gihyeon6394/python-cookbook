# CHAPTER 8 Classes and Objects

이 챕터는 클래스 정의와 관련된 공통 프로그래밍 패턴을 다룬다. special methods, 캡슐레이션, 상속, 메모리 관리, 유용한 디자인 패턴 등을 포함한다.

## 8.1 인스턴스의 문자열 표현 변경

`__repr__()`과 `__str__()`을 정의하여 인스턴스의 출력을 커스팅한다.

```python
class Pair:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __repr__(self):
        return 'Pair({0.x!r}, {0.y!r})'.format(self)   # 재생성 가능한 표현

    def __str__(self):
        return '({0.x!s}, {0.y!s})'.format(self)        # 출력용 표현

>>> p = Pair(3, 4)
>>> p              # __repr__ 호출
Pair(3, 4)
>>> print(p)       # __str__ 호출
(3, 4)
```

`__str__()`이 정의되지 않으면 `__repr__()`이 대체로 사용된다. `__repr__()`의 출력은 가능하면 `eval(repr(x)) == x`를 만족하도록 한다.

## 8.2 문자열 포맷팅 커스팅

`__format__()` 메서드를 정의하면 `format()`과 f-string에서 커스텀 포맷 코드를 사용할 수 있다.

```python
_formats = {
    'ymd': '{d.year}-{d.month}-{d.day}',
    'mdy': '{d.month}/{d.day}/{d.year}',
    'dmy': '{d.day}/{d.month}/{d.year}'
}

class Date:
    def __init__(self, year, month, day):
        self.year, self.month, self.day = year, month, day

    def __format__(self, code):
        if code == '':
            code = 'ymd'
        return _formats[code].format(d=self)

>>> d = Date(2012, 12, 21)
>>> format(d, 'mdy')
'12/21/2012'
>>> 'The date is {:ymd}'.format(d)
'The date is 2012-12-21'
```

## 8.3 Context Management Protocol (with 문)

`__enter__()`과 `__exit__()`를 구현하면 `with` 문과 호환된다. `__exit__()`는 예외 발생 여부와 관계없이 반드시 실행되어 리소스 정리가 보장된다.

```python
from socket import socket, AF_INET, SOCK_STREAM

class LazyConnection:
    def __init__(self, address, family=AF_INET, type=SOCK_STREAM):
        self.address = address
        self.family  = family
        self.type    = type
        self.sock    = None

    def __enter__(self):
        if self.sock is not None:
            raise RuntimeError('Already connected')
        self.sock = socket(self.family, self.type)
        self.sock.connect(self.address)
        return self.sock

    def __exit__(self, exc_ty, exc_val, tb):
        self.sock.close()
        self.sock = None
```

중첩 `with` 문을 지원하려면 단일 소켓 대신 **리스트(스택)**을 사용하여 연결을 관리한다.

```python
class LazyConnection:
    def __init__(self, address, ...):
        self.connections = []          # 스택으로 관리

    def __enter__(self):
        sock = socket(self.family, self.type)
        sock.connect(self.address)
        self.connections.append(sock)
        return sock

    def __exit__(self, exc_ty, exc_val, tb):
        self.connections.pop().close()  # LIFO 순서로 닫기
```

## 8.4 대량 인스턴스 생성 시 메모리 절약 (__slots__)

`__slots__`를 정의하면 인스턴스 내부 표현이 dictionary 대신 고정 크기 배열로 변경되어 메모리가 크게 절약된다.

```python
class Date:
    __slots__ = ['year', 'month', 'day']
    def __init__(self, year, month, day):
        self.year  = year
        self.month = month
        self.day   = day
```

| 구성             | 메모리 (64-bit Python) |
|----------------|---------------------|
| `__slots__` 없음 | ~428 bytes          |
| `__slots__` 사용 | ~156 bytes          |

`__slots__` 사용 시 새로운 속성을 동적으로 추가할 수 없고, 일부 기능(예: 다중 상속)이 제한된다. 백만 건 이상의 인스턴스를 생성하는 데이터 구조에서만 사용하는 것이 권장된다.

## 8.5 클래스 내 이름 캡슐레이션

Python은 언어 레벨의 접근 제한이 없으므로 **이름 규칙**을 따라야 한다.

| 규칙               | 의미                                   | 예시                                       |
|------------------|--------------------------------------|------------------------------------------|
| `_name` (단일 밑줄)  | 내부 구현용. 외부에서 접근하지 않아야 함              | `self._internal`                         |
| `__name` (이중 밑줄) | Name mangling 적용. 상속 시 하위 클래스와 충돌 방지 | `self.__private` → `_ClassName__private` |
| `name_` (끝 밑줄)   | 예약어와의 충돌 회피                          | `lambda_ = 2.0`                          |

이중 밑줄의 주 목적은 **상속에서의 이름 충돌 방지**이다. 단순 캡슐레이션 목적이면 단일 밑줄로 충분하다.

## 8.6 관리된 속성 (Property)

`@property` decorator로 속성의 getter/setter/deleter에 로직을 추가한다.

```python
class Person:
    def __init__(self, first_name):
        self.first_name = first_name   # setter를 통해 초기화

    @property
    def first_name(self):              # getter
        return self._first_name

    @first_name.setter
    def first_name(self, value):       # setter (타입 검증)
        if not isinstance(value, str):
            raise TypeError('Expected a string')
        self._first_name = value

    @first_name.deleter
    def first_name(self):              # deleter
        raise AttributeError("Can't delete attribute")
```

### Computed Property (계산된 속성)

실제로 저장하지 않고 접근 시 동적으로 계산되는 속성도 property로 구현할 수 있다.

```python
import math

class Circle:
    def __init__(self, radius):
        self.radius = radius

    @property
    def area(self):
        return math.pi * self.radius ** 2

    @property
    def perimeter(self):
        return 2 * math.pi * self.radius

>>> c = Circle(4.0)
>>> c.area        # () 없이 속성처럼 접근
50.26548245743669
```

아무 처리도 추가하지 않는 property는 코드를 불필요하게 복잡시키고 속도를 떨어뜨리므로 피하자. 반복 패턴이 많으면 **descriptor**(8.9)를 사용한다.

## 8.7 부모 클래스의 메서드 호출 (super())

`super()`를 사용하여 부모 클래스의 메서드를 호출한다. 다중 상속에서 직접 부모 클래스 이름으로 호출하면 **Base.__init__()이 중복 실행**될 수 있다.

```python
# 잘못된 방식 (다중 상속에서 문제 발생)
class A(Base):
    def __init__(self):
        Base.__init__(self)   # ← 직접 호출

# 올바른 방식
class A(Base):
    def __init__(self):
        super().__init__()    # ← MRO에 따라 올바르게 위임
```

`super()`는 **Method Resolution Order (MRO)**를 기반으로 다음 클래스에 위임한다. MRO는 C3 Linearization으로 결정되며, 모든 클래스가 일관되게 `super()`를
사용하면 각 메서드가 정확히 한 번만 실행된다.

```python
>>> C.__mro__
(<class 'C'>, <class 'A'>, <class 'B'>, <class 'Base'>, <class 'object'>)
```

### 주의사항

같은 이름의 메서드는 호환 가능한 calling signature를 가져야 하고, 최상위 클래스에서 해당 메서드의 구현을 제공해야 한다.

## 8.8 하위 클래스에서 Property 확장

부모의 property를 하위 클래스에서 확장할 때, 한 메서드만 교체하려면 `@ParentClass.name.getter` / `@ParentClass.name.setter` 형태를 사용해야 한다.
`@property`만 사용하면 setter가 사라진다.

```python
class SubPerson(Person):
    @Person.name.getter                      # getter만 교체
    def name(self):
        print('Getting name')
        return super().name

# 또는 setter만 교체
class SubPerson(Person):
    @Person.name.setter
    def name(self, value):
        print('Setting name to', value)
        super(SubPerson, SubPerson).name.__set__(self, value)
```

## 8.9 새로운 종류의 클래스/인스턴스 속성 (Descriptor)

**Descriptor** 클래스를 정의하면 속성 접근(get/set/delete)을 저수준에서 완전히 커스팅할 수 있다. Descriptor는 반드시 **클래스 변수**로 정의해야 한다.

```python
class Integer:
    def __init__(self, name):
        self.name = name

    def __get__(self, instance, cls):
        if instance is None:
            return self
        return instance.__dict__[self.name]

    def __set__(self, instance, value):
        if not isinstance(value, int):
            raise TypeError('Expected an int')
        instance.__dict__[self.name] = value

    def __delete__(self, instance):
        del instance.__dict__[self.name]

class Point:
    x = Integer('x')   # 클래스 변수로 정의
    y = Integer('y')
    def __init__(self, x, y):
        self.x = x      # Integer.__set__ 호출
        self.y = y
```

class decorator와 결합하면 타입 검증을 깔끔하게 적용할 수 있다.

```python
class Typed:
    def __init__(self, name, expected_type):
        self.name = name
        self.expected_type = expected_type
    def __get__(self, instance, cls):
        if instance is None: return self
        return instance.__dict__[self.name]
    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError('Expected ' + str(self.expected_type))
        instance.__dict__[self.name] = value

def typeassert(**kwargs):
    def decorate(cls):
        for name, expected_type in kwargs.items():
            setattr(cls, name, Typed(name, expected_type))
        return cls
    return decorate

@typeassert(name=str, shares=int, price=float)
class Stock:
    def __init__(self, name, shares, price):
        self.name = name
        self.shares = shares
        self.price = price
```

## 8.10 Lazy Property (지연 계산 속성)

속성이 처음 접근될 때만 계산되고, 이후에는 캐시된 값을 반환하는 패턴이다.

```python
class lazyproperty:
    def __init__(self, func):
        self.func = func

    def __get__(self, instance, cls):
        if instance is None:
            return self
        value = self.func(instance)
        setattr(instance, self.func.__name__, value)  # 인스턴스 dict에 저장
        return value

class Circle:
    def __init__(self, radius):
        self.radius = radius

    @lazyproperty
    def area(self):
        print('Computing area')
        return math.pi * self.radius ** 2

>>> c = Circle(4.0)
>>> c.area          # 'Computing area' 출력 후 값 반환
50.265...
>>> c.area          # 출력 없이 캐시된 값 반환
50.265...
```

핵심 원리: `__get__()`만 정의한 descriptor는 **인스턴스 dictionary에 같은 이름의 값이 있으면 호출되지 않는다**. 첫 접근 후 값이 instance dict에 저장되면
descriptor가 우회된다.

## 8.11 데이터 구조 초기화 간소화

공통 base 클래스에 범용 `__init__()`을 정의하여 반복적인 초기화 코드를 줄인다.

```python
class Structure:
    _fields = []
    def __init__(self, *args, **kwargs):
        if len(args) > len(self._fields):
            raise TypeError('Expected {} arguments'.format(len(self._fields)))
        for name, value in zip(self._fields, args):
            setattr(self, name, value)
        for name in self._fields[len(args):]:
            setattr(self, name, kwargs.pop(name))
        if kwargs:
            raise TypeError('Invalid argument(s): {}'.format(','.join(kwargs)))

class Stock(Structure):
    _fields = ['name', 'shares', 'price']

>>> s = Stock('ACME', 50, price=91.1)   # positional + keyword 혼용 가능
```

`setattr()`를 사용해야 한다. `self.__dict__`를 직접 조작하면 `__slots__`이나 property를 사용하는 하위 클래스에서 깨진다.

## 8.12 인터페이스 / Abstract Base Class (ABC)

`abc` 모듈의 `ABCMeta`와 `@abstractmethod`로 직접 인스턴스화할 수 없는 추상 클래스를 정의한다.

```python
from abc import ABCMeta, abstractmethod

class IStream(metaclass=ABCMeta):
    @abstractmethod
    def read(self, maxbytes=-1):
        pass
    @abstractmethod
    def write(self, data):
        pass

# 직접 인스턴스화하면 TypeError 발생
# IStream()  → TypeError

# 구현 클래스는 모든 abstractmethod를 반드시 구현
class SocketStream(IStream):
    def read(self, maxbytes=-1): ...
    def write(self, data): ...

# 기존 클래스를 ABC에 등록도 가능
import io
IStream.register(io.IOBase)
isinstance(open('file.txt'), IStream)   # True
```

`collections` 모듈에는 `Sequence`, `Mapping`, `Iterable` 등 표준 컨테이너 ABC가 제공된다.

## 8.13 데이터 모델 / 타입 시스템 구현

Descriptor와 mixin class를 조합하여 속성 검증 프레임워크를 구축한다.

```python
class Descriptor:
    def __init__(self, name=None, **opts):
        self.name = name
        for key, value in opts.items():
            setattr(self, key, value)
    def __set__(self, instance, value):
        instance.__dict__[self.name] = value

class Typed(Descriptor):
    expected_type = type(None)
    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError('expected ' + str(self.expected_type))
        super().__set__(instance, value)

class Unsigned(Descriptor):
    def __set__(self, instance, value):
        if value < 0:
            raise ValueError('Expected >= 0')
        super().__set__(instance, value)

class MaxSized(Descriptor):
    def __set__(self, instance, value):
        if len(value) >= self.size:
            raise ValueError('size must be < ' + str(self.size))
        super().__set__(instance, value)

# 조합하여 구체적 타입 클래스 생성
class Integer(Typed):         expected_type = int
class UnsignedInteger(Integer, Unsigned): pass
class Float(Typed):           expected_type = float
class UnsignedFloat(Float, Unsigned):     pass
class String(Typed):          expected_type = str
class SizedString(String, MaxSized):      pass

class Stock:
    name   = SizedString('name', size=8)
    shares = UnsignedInteger('shares')
    price  = UnsignedFloat('price')
    def __init__(self, name, shares, price):
        self.name = name; self.shares = shares; self.price = price
```

metaclass를 사용하면 속성 이름을 한 번만 작성하면 된다.

```python
class checkedmeta(type):
    def __new__(cls, clsname, bases, methods):
        for key, value in methods.items():
            if isinstance(value, Descriptor):
                value.name = key           # 자동으로 이름 설정
        return type.__new__(cls, clsname, bases, methods)

class Stock(metaclass=checkedmeta):
    name   = SizedString(size=8)
    shares = UnsignedInteger()
    price  = UnsignedFloat()
```

## 8.14 커스텀 컨테이너 구현

`collections`의 ABC를 상속하면 반드시 구현해야 할 메서드를 강제받고, 나머지 메서드는 자동으로 제공된다.

```python
import collections, bisect

class SortedItems(collections.Sequence):
    def __init__(self, initial=None):
        self._items = sorted(initial) if initial else []

    def __getitem__(self, index):       # 필수 구현
        return self._items[index]

    def __len__(self):                  # 필수 구현
        return len(self._items)

    def add(self, item):
        bisect.insort(self._items, item)

>>> items = SortedItems([5, 1, 3])
>>> list(items)       # [1, 3, 5]
>>> items.add(2)
>>> 2 in items        # True (자동 제공)
>>> items[1:3]        # slicing도 자동 지원
```

`MutableSequence`를 상속하면 `append()`, `remove()`, `count()` 등도 자동으로 제공된다.

## 8.15 속성 접근 위임 (Delegation)

`__getattr__()`를 정의하여 내부 객체에 속성 접근을 위임할 수 있다. Proxy 패턴에서 주로 사용한다.

```python
class Proxy:
    def __init__(self, obj):
        self._obj = obj

    def __getattr__(self, name):         # 속성 조회 위임
        print('getattr:', name)
        return getattr(self._obj, name)

    def __setattr__(self, name, value):  # 속성 할당 위임
        if name.startswith('_'):
            super().__setattr__(name, value)   # Proxy 자신의 속성
        else:
            setattr(self._obj, name, value)    # 내부 객체에 위임

    def __delattr__(self, name):
        if name.startswith('_'):
            super().__delattr__(name)
        else:
            delattr(self._obj, name)
```

**주의:** `__getattr__()`는 이중 밑줄로 둘러싸인 special method(`__len__`, `__getitem__` 등)에 적용되지 않는다. 이런 메서드는 별도로 직접 정의해야 한다.

## 8.16 클래스에 여러 생성자 정의

`@classmethod`를 사용하여 대체 생성자를 정의한다. `cls` 인수를 사용하여 상속해도 올바르게 동작한다.

```python
import time

class Date:
    def __init__(self, year, month, day):
        self.year, self.month, self.day = year, month, day

    @classmethod
    def today(cls):                        # 대체 생성자
        t = time.localtime()
        return cls(t.tm_year, t.tm_mon, t.tm_mday)

a = Date(2012, 12, 21)   # 기본 생성자
b = Date.today()          # 대체 생성자
```

## 8.17 __init__() 호출 없이 인스턴스 생성

`__new__()`를 직접 호출하면 `__init__()`을 우회할 수 있다. deserialization이나 대체 생성자에서 활용한다.

```python
class Date:
    def __init__(self, year, month, day):
        self.year, self.month, self.day = year, month, day

# __init__ 우회
d = Date.__new__(Date)
# setattr로 속성 설정 (__dict__ 직접 조작 피해야 함)
for key, value in {'year': 2012, 'month': 8, 'day': 29}.items():
    setattr(d, key, value)
```

## 8.18 Mixin 클래스로 기능 확장

Mixin은 다중 상속을 통해 기존 클래스에 기능을 추가하는 재사용 가능한 클래스이다.

```python
class LoggedMappingMixin:
    __slots__ = ()                          # 상태 없음
    def __getitem__(self, key):
        print('Getting ' + str(key))
        return super().__getitem__(key)     # MRO 다음 클래스로 위임
    def __setitem__(self, key, value):
        print('Setting {} = {!r}'.format(key, value))
        return super().__setitem__(key, value)

class SetOnceMappingMixin:
    __slots__ = ()
    def __setitem__(self, key, value):
        if key in self:
            raise KeyError(str(key) + ' already set')
        return super().__setitem__(key, value)

# 조합 사용
class LoggedDict(LoggedMappingMixin, dict):
    pass

class StringOrderedDict(StringKeysMappingMixin, SetOnceMappingMixin, OrderedDict):
    pass
```

### Mixin 설계 원칙

직접 인스턴스화하지 않는다. `__slots__ = ()`로 상태 없음을 선언한다. `super()`를 반드시 사용하여 MRO에 따라 올바르게 위임한다. `__init__()`이 필요하면 keyword-only
인수로 고유 이름을 사용하고 `super().__init__(*args, **kwargs)`를 호출한다.

## 8.19 상태 기계 (State Machine) 구현

각 상태를 별도 클래스로 분리하여 조건문 없이 동작을 구현한다.

```python
class Connection:
    def __init__(self):
        self.new_state(ClosedConnectionState)
    def new_state(self, newstate):
        self._state = newstate
    def read(self):
        return self._state.read(self)
    def write(self, data):
        return self._state.write(self, data)
    def open(self):
        return self._state.open(self)
    def close(self):
        return self._state.close(self)

class ClosedConnectionState:
    @staticmethod
    def read(conn):  raise RuntimeError('Not open')
    @staticmethod
    def write(conn, data):  raise RuntimeError('Not open')
    @staticmethod
    def open(conn):  conn.new_state(OpenConnectionState)
    @staticmethod
    def close(conn): raise RuntimeError('Already closed')

class OpenConnectionState:
    @staticmethod
    def read(conn):  print('reading')
    @staticmethod
    def write(conn, data): print('writing')
    @staticmethod
    def open(conn):  raise RuntimeError('Already open')
    @staticmethod
    def close(conn): conn.new_state(ClosedConnectionState)
```

더 간결한 대안으로 `self.__class__`를 직접 교체하는 방식도 있다. 이 경우 위임 단계가 없어져 속도가 빨라진다.

```python
class Connection:
    def new_state(self, newstate):
        self.__class__ = newstate          # 인스턴스의 클래스 자체 교체
    ...

class ClosedConnection(Connection):
    def open(self):
        self.new_state(OpenConnection)
    ...
```

## 8.20 문자열로 메서드 호출

`getattr()`로 문자열 이름으로 메서드를 조회하여 호출한다. `operator.methodcaller()`는 동일 인수로 반복 호출할 때 유용하다.

```python
import operator

p = Point(2, 3)
d = getattr(p, 'distance')(0, 0)                  # p.distance(0, 0)

# 리스트 정렬 시 활용
points.sort(key=operator.methodcaller('distance', 0, 0))
```

## 8.21 Visitor 패턴 구현

복잡한 트리 구조를 순회할 때, 노드 타입에 따라 다른 처리를 하는 패턴이다. `getattr()`로 메서드 이름을 동적 생성하여 dispatch한다.

```python
class NodeVisitor:
    def visit(self, node):
        methname = 'visit_' + type(node).__name__
        meth = getattr(self, methname, self.generic_visit)
        return meth(node)

    def generic_visit(self, node):
        raise RuntimeError('No {} method'.format('visit_' + type(node).__name__))

# 수학 표현식 평가 예시
class Evaluator(NodeVisitor):
    def visit_Number(self, node):
        return node.value
    def visit_Add(self, node):
        return self.visit(node.left) + self.visit(node.right)
    def visit_Sub(self, node):
        return self.visit(node.left) - self.visit(node.right)
    def visit_Mul(self, node):
        return self.visit(node.left) * self.visit(node.right)
    def visit_Div(self, node):
        return self.visit(node.left) / self.visit(node.right)
```

## 8.22 재귀 없는 Visitor 패턴

깊은 트리 구조에서 재귀 Visitor는 recursion depth 제한에 걸린다. **generator와 스택**을 활용하면 재귀를 제거한다.

```python
import types

class NodeVisitor:
    def visit(self, node):
        stack = [node]
        last_result = None
        while stack:
            try:
                last = stack[-1]
                if isinstance(last, types.GeneratorType):
                    stack.append(last.send(last_result))
                    last_result = None
                elif isinstance(last, Node):
                    stack.append(self._visit(stack.pop()))
                else:
                    last_result = stack.pop()
            except StopIteration:
                stack.pop()
        return last_result
```

visitor 메서드에서 재귀 호출 대신 `yield`를 사용한다:

```python
class Evaluator(NodeVisitor):
    def visit_Number(self, node):
        return node.value                                    # return 사용
    def visit_Add(self, node):
        yield (yield node.left) + (yield node.right)        # yield로 대체
    def visit_Sub(self, node):
        yield (yield node.left) - (yield node.right)
```

`yield node.left`는 해당 노드를 스택에 ` push`하고, 처리 결과를 `send()`로 다시 받아든다. 동일한 visitor 클래스로 재귀·비재귀 모두 사용 가능하다.

## 8.23 순환 데이터 구조와 메모리 관리

부모↔자식 등 순환 참조가 있는 구조는 참조 카운팅으로 해제되지 않는다. **weakref**를 사용하여 순환을 단절시킨다.

```python
import weakref

class Node:
    def __init__(self, value):
        self.value = value
        self._parent = None
        self.children = []

    @property
    def parent(self):
        return self._parent if self._parent is None else self._parent()

    @parent.setter
    def parent(self, node):
        self._parent = weakref.ref(node)   # 약한 참조로 저장

    def add_child(self, child):
        self.children.append(child)
        child.parent = self

>>> root = Node('parent')
>>> c1 = Node('child')
>>> root.add_child(c1)
>>> del root
>>> print(c1.parent)   # None (약한 참조이므로 자동 무효화)
```

`__del__()`을 정의한 객체가 순환에 포함되면 **영원히 가비지 컬렉트되지 않아 메모리 누수**가 발생한다.

## 8.24 비교 연산자 지원 (total_ordering)

`@functools.total_ordering`을 사용하면 `__eq__()`과 비교 메서드 하나만 정의하면 나머지가 자동 생성된다.

```python
from functools import total_ordering

@total_ordering
class House:
    def __init__(self, name, style):
        self.name  = name
        self.style = style
        self.rooms = []

    @property
    def living_space_footage(self):
        return sum(r.square_feet for r in self.rooms)

    def __eq__(self, other):
        return self.living_space_footage == other.living_space_footage

    def __lt__(self, other):              # 이 하나만 정의하면 됨
        return self.living_space_footage < other.living_space_footage

# __le__, __gt__, __ge__, __ne__ 자동 생성
```

## 8.25 캐시된 인스턴스 생성

동일 인수로 생성된 인스턴스를 재사용하려면 **팩토리 함수 + WeakValueDictionary**를 사용한다.

```python
import weakref

class Spam:
    def __init__(self, name):
        self.name = name

_spam_cache = weakref.WeakValueDictionary()

def get_spam(name):
    if name not in _spam_cache:
        s = Spam(name)
        _spam_cache[name] = s
    else:
        s = _spam_cache[name]
    return s

>>> a = get_spam('foo')
>>> c = get_spam('foo')
>>> a is c            # True (동일 인스턴스)
```

`WeakValueDictionary`는 해당 인스턴스가 다른 곳에서 참조되지 않으면 자동으로 캐시에서 제거된다. `__new__()`를 오버라이드하는 방식은 `__init__()`이 매번 실행되어 적합하지 않다.