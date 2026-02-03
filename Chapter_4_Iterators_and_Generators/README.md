# CHAPTER 4 Iterators and Generators

Iteration은 Python의 강력한 기능 중 하나로, 단순한 시퀀스 처리를 넘어 커스텀 iterator 객체 생성, itertools 모듈의 활용, generator function 등 다양한 패턴을 지원한다.

## 4.1 Iterator를 수동으로 소비하기

`for` 루프 대신 수동으로 iterable을 처리하려면 `next()` 함수를 사용하고 `StopIteration` 예외를 처리한다.

```python
with open('/etc/passwd') as f:
    try:
        while True:
            line = next(f)
            print(line, end='')
    except StopIteration:
        pass
```

`next()`에 기본값을 넘기면 `StopIteration` 대신 해당 값을 반환할 수 있다.

```python
with open('/etc/passwd') as f:
    while True:
        line = next(f, None)
        if line is None:
            break
        print(line, end='')
```

iteration의 기본 메커니즘은 다음과 같다. `iter()`가 `__iter__()`를 호출하여 iterator를 반환하고, `next()`가 `__next__()`를 호출하여 값을 순차적으로 생성한다. 모든
값이 소진되면 `StopIteration`이 발생한다.

```python
>>> items = [1, 2, 3]
>>> it = iter(items)       # items.__iter__() 호출
>>> next(it)               # it.__next__() 호출
1
>>> next(it)
2
>>> next(it)
3
>>> next(it)
StopIteration
```

## 4.2 Iteration 위임 (Delegating Iteration)

커스텀 컨테이너 객체에 iteration을 지원하려면 `__iter__()` 메서드를 정의하여 내부 컨테이너에 위임하면 된다.

```python
class Node:
    def __init__(self, value):
        self._value = value
        self._children = []

    def __repr__(self):
        return 'Node({!r})'.format(self._value)

    def add_child(self, node):
        self._children.append(node)

    def __iter__(self):
        return iter(self._children)

# 사용 예시
root = Node(0)
root.add_child(Node(1))
root.add_child(Node(2))
for ch in root:
    print(ch)  # Node(1), Node(2) 출력
```

`iter(s)`는 `s.__iter__()`를 호출한 후 underlying iterator를 반환하는 단순한 shortcut이다.

## 4.3 Generator를 이용한 새로운 Iteration 패턴

기존 내장 함수(`range()`, `reversed()` 등)와 다른 커스텀 iteration 패턴을 만들려면 **generator function**을 사용한다. `yield` 키워드가 포함된 함수는 자동으로
generator가 된다.

```python
def frange(start, stop, increment):
    x = start
    while x < stop:
        yield x
        x += increment

>>> list(frange(0, 1, 0.125))
[0, 0.125, 0.25, 0.375, 0.5, 0.625, 0.75, 0.875]
```

generator는 iteration이 실행될 때만 실행된다. 아래 예시로 그 메커니즘을 확인할 수 있다.

```python
>>> def countdown(n):
...     print('Starting to count from', n)
...     while n > 0:
...         yield n
...         n -= 1
...     print('Done!')

>>> c = countdown(3)       # 생성 시 출력 없음
>>> next(c)                # 첫 번째 yield까지 실행
Starting to count from 3
3
>>> next(c)
2
>>> next(c)
1
>>> next(c)                # 반복 종료
Done!
StopIteration
```

## 4.4 Iterator Protocol 구현

커스텀 객체에 iteration을 지원하는 가장 간단한 방법은 **generator function**을 사용하는 것이다. 아래는 트리 구조의 depth-first 탐색 예시이다.

```python
class Node:
    def __init__(self, value):
        self._value = value
        self._children = []

    def __repr__(self):
        return 'Node({!r})'.format(self._value)

    def add_child(self, node):
        self._children.append(node)

    def __iter__(self):
        return iter(self._children)

    def depth_first(self):
        yield self
        for c in self:
            yield from c.depth_first()

# 사용 예시
root = Node(0)
child1, child2 = Node(1), Node(2)
root.add_child(child1)
root.add_child(child2)
child1.add_child(Node(3))
child1.add_child(Node(4))
child2.add_child(Node(5))

for ch in root.depth_first():
    print(ch)
# 출력: Node(0), Node(1), Node(3), Node(4), Node(2), Node(5)
```

`yield from`을 사용하면 depth-first 탐색이 간결하게 표현된다. 이를 별도의 `DepthFirstIterator` 클래스로 구현하면 복잡한 state 관리가 필요하여 유지하기 어렵다. *
*generator function을 사용하는 것이 권장된다.**

## 4.5 역순 Iteration

`reversed()` 내장 함수를 사용한다. 단, 객체의 크기가 알려져 있거나 `__reversed__()` 메서드가 구현되어 있어야 한다. 둘 다 불가하면 먼저 `list`로 변환해야 한다.

```python
# 파일을 역순으로 출력
f = open('somefile')
for line in reversed(list(f)):
    print(line, end='')
```

커스텀 클래스에서 `__reversed__()`를 직접 정의하면 list 변환 없이 메모리 효율적으로 역순 iteration이 가능하다.

```python
class Countdown:
    def __init__(self, start):
        self.start = start

    def __iter__(self):
        n = self.start
        while n > 0:
            yield n
            n -= 1

    def __reversed__(self):
        n = 1
        while n <= self.start:
            yield n
            n += 1
```

## 4.6 Extra State를 가진 Generator Function

generator가 외부에 상태를 노출해야 한다면, **클래스로 구현**하고 `__iter__()` 메서드에 generator 로직을 넣는다.

```python
from collections import deque

class linehistory:
    def __init__(self, lines, histlen=3):
        self.lines = lines
        self.history = deque(maxlen=histlen)

    def __iter__(self):
        for lineno, line in enumerate(self.lines, 1):
            self.history.append((lineno, line))
            yield line

    def clear(self):
        self.history.clear()

# 사용 예시
with open('somefile.txt') as f:
    lines = linehistory(f)
    for line in lines:
        if 'python' in line:
            for lineno, hline in lines.history:
                print('{}:{}'.format(lineno, hline), end='')
```

`for` 루프가 아닌 다른 방식으로 iteration을 구현하려면 먼저 `iter()`를 호출해야 한다.

```python
>>> lines = linehistory(f)
>>> it = iter(lines)       # iter() 호출 필수
>>> next(it)
'hello world\n'
```

## 4.7 Iterator의 Slice 가져오기

iterator와 generator는 일반 slicing 연산자가 불가하므로 `itertools.islice()`를 사용한다.

```python
import itertools

def count(n):
    while True:
        yield n
        n += 1

c = count(0)
for x in itertools.islice(c, 10, 20):
    print(x)   # 10 ~ 19 출력
```

`islice()`는 시작 인덱스까지의 모든 항목을 소비하고 버린다. iterator는 되돌리기가 불가하므로 이 점을 주의해야 한다.

## 4.8 Iterable의 앞부분 건너뛰기

### dropwhile 활용

조건을 만족하는 앞부분을 건너뛰려면 `itertools.dropwhile()`를 사용한다.

```python
from itertools import dropwhile

with open('/etc/passwd') as f:
    for line in dropwhile(lambda line: line.startswith('#'), f):
        print(line, end='')
```

건너뛰는 횟수를 정확히 알면 `islice()`를 사용할 수 있다.

```python
from itertools import islice

items = ['a', 'b', 'c', 1, 4, 10, 15]
for x in islice(items, 3, None):   # 처음 3개 건너뜀
    print(x)   # 1, 4, 10, 15 출력
```

`dropwhile()`과 단순 filtering의 차이점은 중요하다. `dropwhile()`은 조건이 False가 되는 순간 이후의 모든 항목을 반환하지만, filtering은 파일 전체에서 조건을 만족하지 않는
행을 모두 제거한다.

## 4.9 모든 조합(Combinations)과 순열(Permutations) 반복

`itertools` 모듈의 세 가지 함수를 활용한다.

| 함수                                        | 설명                   | 예시                                                                              |
|-------------------------------------------|----------------------|---------------------------------------------------------------------------------|
| `permutations(items, r)`                  | 모든 순열 생성 (순서 구분)     | `permutations(['a','b','c'], 2)` → `('a','b'), ('a','c'), ...`                  |
| `combinations(items, r)`                  | 조합 생성 (순서 무시, 중복 없음) | `combinations(['a','b','c'], 2)` → `('a','b'), ('a','c'), ('b','c')`            |
| `combinations_with_replacement(items, r)` | 조합 생성 (중복 허용)        | `combinations_with_replacement(['a','b','c'], 2)` → `('a','a'), ('a','b'), ...` |

복잡한 iteration 문제에 직면하면 항상 `itertools`를 먼저 살펴보는 것이 좋다.

## 4.10 Index-Value 쌍으로 Iteration

`enumerate()` 내장 함수를 사용하여 인덱스와 값을 동시에 가져온다. `start` 인수로 시작 번호를 지정할 수 있다.

```python
my_list = ['a', 'b', 'c']
for idx, val in enumerate(my_list, 1):   # 1부터 시작
    print(idx, val)
# 1 a, 2 b, 3 c
```

파일에서 단어가 어떤 줄에 등장하는지 추적하는 것도 가능하다.

```python
from collections import defaultdict

word_summary = defaultdict(list)
with open('myfile.txt', 'r') as f:
    lines = f.readlines()
for idx, line in enumerate(lines):
    words = [w.strip().lower() for w in line.split()]
    for word in words:
        word_summary[word].append(idx)
```

tuple이 포함된 시퀀스에서 unpacking과 함께 사용할 때는 주의이다.

```python
data = [(1, 2), (3, 4), (5, 6)]
for n, (x, y) in enumerate(data):   # ✓ 올바름
    ...
for n, x, y in enumerate(data):     # ✗ 오류
    ...
```

## 4.11 여러 시퀀스를 동시에 Iteration

`zip()` 함수를 사용하여 여러 시퀀스를 동시에 순회한다. 가장 짧은 시퀀스 길이를 기준으로 종료된다.

```python
headers = ['name', 'shares', 'price']
values  = ['ACME', 100, 490.1]
s = dict(zip(headers, values))   # {'name':'ACME', 'shares':100, 'price':490.1}
```

길이가 다른 시퀀스에서 짧은 쪽의 빈 값을 채우려면 `itertools.zip_longest()`를 사용한다.

```python
from itertools import zip_longest

a = [1, 2, 3]
b = ['w', 'x', 'y', 'z']
for i in zip_longest(a, b, fillvalue=0):
    print(i)
# (1, 'w'), (2, 'x'), (3, 'y'), (0, 'z')
```

`zip()`은 iterator를 반환하므로 리스트로 저장하려면 `list(zip(a, b))`를 사용해야 한다.

## 4.12 서로 다른 컨테이너의 항목 Iteration

여러 컨테이너의 항목을 하나로 순회하려면 `itertools.chain()`을 사용한다. 중첩 루프 없이 깔끔하게 처리한다.

```python
from itertools import chain

active_items = set()
inactive_items = set()

for item in chain(active_items, inactive_items):
    # 모든 항목 처리
    ...
```

`chain()`은 `a + b` 같은 시퀀스 합치기보다 메모리 효율적이다. `a + b`는 새로운 시퀀스를 생성하고 같은 타입이어야 하지만, `chain()`은 그런 제약이 없다.

## 4.13 데이터 처리 Pipeline 생성

메모리에 전체 데이터를 올릴 수 없는 대량 데이터 처리에 **generator function**을 이용한 pipeline 패턴을 사용한다.

각 단계를 작은 generator function으로 분리하여 구성한다.

```python
import os, fnmatch, gzip, bz2, re

def gen_find(filepat, top):
    for path, dirlist, filelist in os.walk(top):
        for name in fnmatch.filter(filelist, filepat):
            yield os.path.join(path, name)

def gen_opener(filenames):
    for filename in filenames:
        if filename.endswith('.gz'):
            f = gzip.open(filename, 'rt')
        elif filename.endswith('.bz2'):
            f = bz2.open(filename, 'rt')
        else:
            f = open(filename, 'rt')
        yield f
        f.close()

def gen_concatenate(iterators):
    for it in iterators:
        yield from it

def gen_grep(pattern, lines):
    pat = re.compile(pattern)
    for line in lines:
        if pat.search(line):
            yield line
```

Pipeline을 조합하면 다음과 같다.

```python
lognames  = gen_find('access-log*', 'www')
files     = gen_opener(lognames)
lines     = gen_concatenate(files)
pylines   = gen_grep('(?i)python', lines)

for line in pylines:
    print(line)
```

generator expression을 추가하여 pipeline을 확장할 수 있다.

```python
bytecolumn = (line.rsplit(None, 1)[1] for line in pylines)
bytes      = (int(x) for x in bytecolumn if x != '-')
print('Total', sum(bytes))
```

이 패턴의 장점은 각 generator가 작고 독립적이어 유지하기 쉽고, 재사용 가능하며, 메모리 효율이 높다. `yield`는 데이터 생산자, `for` 루프는 데이터 소비자로 작동한다.

`gen_concatenate()`에서 `itertools.chain(*files)`를 사용하지 않는 이유는, 그렇게 하면 `gen_opener()` generator가 즉시 완전히 소비되어 파일이 제대로 닫히지
않기 때문이다.

## 4.14 중첩 시퀀스를 평탄화 (Flattening)

재귀적 generator function과 `yield from`을 사용하여 중첩 시퀀스를 단일 리스트로 평탄화한다.

```python
from collections.abc import Iterable

def flatten(items, ignore_types=(str, bytes)):
    for x in items:
        if isinstance(x, Iterable) and not isinstance(x, ignore_types):
            yield from flatten(x)
        else:
            yield x

items = [1, 2, [3, 4, [5, 6], 7], 8]
for x in flatten(items):
    print(x)   # 1 2 3 4 5 6 7 8
```

`ignore_types=(str, bytes)`는 문자열과 bytes가 개별 문자로 분해되지 않도록 방지한다.

```python
>>> items = ['Dave', 'Paula', ['Thomas', 'Lewis']]
>>> list(flatten(items))
['Dave', 'Paula', 'Thomas', 'Lewis']
```

## 4.15 정렬된 Iterable들을 병합하여 정렬 순회

여러 정렬된 시퀀스를 하나의 정렬된 순서로 순회하려면 `heapq.merge()`를 사용한다.

```python
import heapq

a = [1, 4, 7, 10]
b = [2, 5, 6, 11]
for c in heapq.merge(a, b):
    print(c)   # 1, 2, 4, 5, 6, 7, 10, 11
```

`heapq.merge()`는 이터레이티브로 작동하여 전체 시퀀스를 한 번에 읽지 않으므로 매우 큰 데이터에도 적합하다. 단, **입력 시퀀스가 이미 정렬되어 있어야** 한다. 정렬 여부에 대한 검증은 수행하지
않는다.

## 4.16 무한 while 루프를 Iterator로 대체

반복적으로 함수를 호출하는 `while` 루프를 `iter()`의 sentinel 기능으로 교체할 수 있다.

```python
# 기존 방식
CHUNKSIZE = 8192
def reader(s):
    while True:
        data = s.recv(CHUNKSIZE)
        if data == b'':
            break
        process_data(data)

# iter() + sentinel로 개선
def reader(s):
    for chunk in iter(lambda: s.recv(CHUNKSIZE), b''):
        process_data(chunk)
```

`iter(callable, sentinel)` 형태로 사용하면, `callable`을 반복 호출하여 반환값이 `sentinel`과 일치할 때까지 값을 생성하는 iterator를 만든다. I/O 작업에서 반복적인
`read()` 또는 `recv()` 호출과 종료 조건 검사를 하나의 구문으로 깔끔하게 정리할 수 있다.