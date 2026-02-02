# CHAPTER 1 Data Structures and Algorithms

Python은 list, set, dictionary와 같은 다양한 유용한 내장 데이터 구조를 제공한다. 이 장의 목표는 데이터와 관련된 일반적인 데이터 구조와 알고리즘을 논의하는 것이다. 또한
`collections` 모듈에 포함된 다양한 데이터 구조에 대해서도 다룬다.

## 1.1. Unpacking a Sequence into Separate Variables

### 문제

N개의 요소를 가진 tuple이나 sequence를 N개의 변수로 unpack하고 싶다.

### 해법

간단한 할당 연산으로 sequence(또는 iterable)를 변수로 unpack할 수 있다. 변수의 수와 구조가 sequence와 일치해야 한다.

```python
>>> p = (4, 5)
>>> x, y = p
>>> x
4
>>> y
5

>>> data = [ 'ACME', 50, 91.1, (2012, 12, 21) ]
>>> name, shares, price, date = data
>>> name
'ACME'
>>> date
(2012, 12, 21)

>>> name, shares, price, (year, mon, day) = data
>>> year
2012
>>> day
21
```

### 논의

Unpacking은 tuple이나 list뿐만 아니라 string, file, iterator, generator 등 iterable한 모든 객체에 작동한다.

```python
>>> s = 'Hello'
>>> a, b, c, d, e = s
>>> a
'H'
>>> e
'o'
```

특정 값을 버리고 싶을 때는 사용하지 않는 변수명(예: `_`)을 사용할 수 있다.

```python
>>> data = [ 'ACME', 50, 91.1, (2012, 12, 21) ]
>>> _, shares, price, _ = data
>>> shares
50
```

## 1.2. Unpacking Elements from Iterables of Arbitrary Length

### 문제

N개보다 긴 iterable에서 N개 요소를 unpack할 때 "too many values to unpack" 예외가 발생한다.

### 해법

Python의 "star expression"을 사용한다.

```python
def drop_first_last(grades):
    first, *middle, last = grades
    return avg(middle)

>>> record = ('Dave', 'dave@example.com', '773-555-1212', '847-555-1212')
>>> name, email, *phone_numbers = record
>>> phone_numbers
['773-555-1212', '847-555-1212']
```

Star 변수는 항상 list이며, 첫 번째 위치에도 올 수 있다.

```python
>>> *trailing, current = [10, 8, 7, 1, 9, 5, 10, 3]
>>> trailing
[10, 8, 7, 1, 9, 5, 10]
>>> current
3
```

### 논의

Star unpacking은 가변 길이 tuple sequence를 반복할 때 특히 유용하다.

```python
records = [
    ('foo', 1, 2),
    ('bar', 'hello'),
    ('foo', 3, 4),
]

for tag, *args in records:
    if tag == 'foo':
        do_foo(*args)
    elif tag == 'bar':
        do_bar(*args)
```

String 처리와 결합할 때도 유용하다.

```python
>>> line = 'nobody:*:-2:-2:Unprivileged User:/var/empty:/usr/bin/false'
>>> uname, *fields, homedir, sh = line.split(':')
>>> uname
'nobody'
>>> homedir
'/var/empty'
```

## 1.3. Keeping the Last N Items

### 문제

반복이나 처리 중 마지막 몇 개 아이템의 제한된 이력을 유지하고 싶다.

### 해법

`collections.deque`를 사용한다.

```python
from collections import deque

def search(lines, pattern, history=5):
    previous_lines = deque(maxlen=history)
    for line in lines:
        if pattern in line:
            yield line, previous_lines
        previous_lines.append(line)
```

### 논의

`deque(maxlen=N)`은 고정 크기 큐를 생성한다. 큐가 가득 찬 상태에서 새 항목이 추가되면 가장 오래된 항목이 자동으로 제거된다.

```python
>>> q = deque(maxlen=3)
>>> q.append(1)
>>> q.append(2)
>>> q.append(3)
>>> q
deque([1, 2, 3], maxlen=3)
>>> q.append(4)
>>> q
deque([2, 3, 4], maxlen=3)
```

Maximum size를 지정하지 않으면 unbounded queue가 된다.

```python
>>> q = deque()
>>> q.append(1)
>>> q.appendleft(4)
>>> q
deque([4, 1, 2, 3])
>>> q.pop()
3
>>> q.popleft()
4
```

Queue의 양쪽 끝에서 항목 추가/제거는 O(1) 복잡도를 가진다. List의 앞쪽 삽입/제거가 O(N)인 것과 대조적이다.

## 1.4. Finding the Largest or Smallest N Items

### 문제

컬렉션에서 가장 큰 N개 또는 가장 작은 N개 항목의 리스트를 만들고 싶다.

### 해법

`heapq` 모듈의 `nlargest()`와 `nsmallest()` 함수를 사용한다.

```python
import heapq
nums = [1, 8, 2, 23, 7, -4, 18, 23, 42, 37, 2]
print(heapq.nlargest(3, nums))  # [42, 37, 23]
print(heapq.nsmallest(3, nums))  # [-4, 1, 2]
```

`key` 매개변수로 복잡한 데이터 구조에도 사용할 수 있다.

```python
portfolio = [
    {'name': 'IBM', 'shares': 100, 'price': 91.1},
    {'name': 'AAPL', 'shares': 50, 'price': 543.22},
    {'name': 'FB', 'shares': 200, 'price': 21.09},
]
cheap = heapq.nsmallest(3, portfolio, key=lambda s: s['price'])
expensive = heapq.nlargest(3, portfolio, key=lambda s: s['price'])
```

### 논의

Heap의 가장 중요한 특징은 `heap[0]`이 항상 가장 작은 항목이라는 것이다. `heapq.heappop()` 메서드로 다음 항목을 쉽게 찾을 수 있다(O(log N) 연산).

```python
>>> nums = [1, 8, 2, 23, 7, -4, 18, 23, 42, 37, 2]
>>> heap = list(nums)
>>> heapq.heapify(heap)
>>> heapq.heappop(heap)
-4
>>> heapq.heappop(heap)
1
```

**성능 고려사항:**

- N이 컬렉션 크기에 비해 작을 때 이 함수들이 우수한 성능을 제공한다
- N=1이면 `min()`과 `max()`가 더 빠르다
- N이 컬렉션 크기와 비슷하면 정렬 후 슬라이싱이 더 빠르다: `sorted(items)[:N]` 또는 `sorted(items)[-N:]`

## 1.5. Implementing a Priority Queue

### 문제

주어진 우선순위로 항목을 정렬하고 각 pop 연산마다 항상 가장 높은 우선순위의 항목을 반환하는 큐를 구현하고 싶다.

### 해법

`heapq` 모듈을 사용한 priority queue 구현:

```python
import heapq

class PriorityQueue:
    def __init__(self):
        self._queue = []
        self._index = 0
    
    def push(self, item, priority):
        heapq.heappush(self._queue, (-priority, self._index, item))
        self._index += 1
    
    def pop(self):
        return heapq.heappop(self._queue)[-1]
```

사용 예:

```python
>>> q = PriorityQueue()
>>> q.push(Item('foo'), 1)
>>> q.push(Item('bar'), 5)
>>> q.push(Item('spam'), 4)
>>> q.push(Item('grok'), 1)
>>> q.pop()
Item('bar')
>>> q.pop()
Item('spam')
```

### 논의

Queue는 `(-priority, index, item)` 형태의 tuple로 구성된다. Priority 값을 음수로 만들어 가장 높은 우선순위부터 가장 낮은 우선순위 순으로 정렬한다(heap은 기본적으로
최소값부터 정렬).

Index 변수의 역할:

- 같은 우선순위를 가진 항목들을 삽입 순서대로 정렬한다
- 비교 연산이 작동하도록 만든다(같은 우선순위일 때 Item 객체끼리 비교하면 에러 발생)

## 1.6. Mapping Keys to Multiple Values in a Dictionary

### 문제

키를 여러 값에 매핑하는 dictionary("multidict")를 만들고 싶다.

### 해법

여러 값을 list나 set 같은 다른 container에 저장한다.

```python
d = {
    'a' : [1, 2, 3],
    'b' : [4, 5]
}
e = {
    'a' : {1, 2, 3},
    'b' : {4, 5}
}
```

`collections.defaultdict`를 사용하면 쉽게 구성할 수 있다.

```python
from collections import defaultdict

d = defaultdict(list)
d['a'].append(1)
d['a'].append(2)
d['b'].append(4)

d = defaultdict(set)
d['a'].add(1)
d['a'].add(2)
d['b'].add(4)
```

### 논의

일반 dictionary에서 `setdefault()`를 사용할 수도 있지만, 많은 프로그래머가 `setdefault()`를 부자연스럽게 느낀다.

```python
d = {}
d.setdefault('a', []).append(1)
d.setdefault('a', []).append(2)
```

`defaultdict`를 사용하면 코드가 훨씬 깔끔해진다.

## 1.7. Keeping Dictionaries in Order

### 문제

Dictionary를 생성하고, 반복이나 직렬화 시 항목의 순서를 제어하고 싶다.

### 해법

`collections.OrderedDict`를 사용한다. 데이터의 원래 삽입 순서를 정확히 보존한다.

```python
from collections import OrderedDict
d = OrderedDict()
d['foo'] = 1
d['bar'] = 2
d['spam'] = 3
d['grok'] = 4

for key in d:
    print(key, d[key])
# 출력: "foo 1", "bar 2", "spam 3", "grok 4"
```

JSON 인코딩에서 필드 순서를 정확히 제어하고 싶을 때 특히 유용하다.

```python
>>> import json
>>> json.dumps(d)
'{"foo": 1, "bar": 2, "spam": 3, "grok": 4}'
```

### 논의

`OrderedDict`는 내부적으로 doubly linked list를 유지하여 삽입 순서에 따라 키를 정렬한다. `OrderedDict`의 크기는 추가 linked list로 인해 일반 dictionary의 두
배 이상이다.

## 1.8. Calculating with Dictionaries

### 문제

Dictionary 데이터에 대해 다양한 계산(최솟값, 최댓값, 정렬 등)을 수행하고 싶다.

### 해법

`zip()`을 사용하여 dictionary의 키와 값을 반전시킨다.

```python
prices = {
    'ACME': 45.23,
    'AAPL': 612.78,
    'IBM': 205.55,
    'HPQ': 37.20,
    'FB': 10.75
}

min_price = min(zip(prices.values(), prices.keys()))
# min_price is (10.75, 'FB')

max_price = max(zip(prices.values(), prices.keys()))
# max_price is (612.78, 'AAPL')

prices_sorted = sorted(zip(prices.values(), prices.keys()))
# [(10.75, 'FB'), (37.2, 'HPQ'), (45.23, 'ACME'), ...]
```

### 논의

`zip()`은 한 번만 소비할 수 있는 iterator를 생성한다.

Dictionary에서 직접 reduction을 수행하면 키만 처리된다:

```python
min(prices)  # Returns 'AAPL'
max(prices)  # Returns 'IBM'
```

`key` 함수를 사용할 수도 있지만, 추가 lookup이 필요하다:

```python
min(prices, key=lambda k: prices[k])  # Returns 'FB'
```

`zip()` 솔루션은 dictionary를 `(value, key)` 쌍의 sequence로 "반전"시켜 문제를 해결한다.

## 1.9. Finding Commonalities in Two Dictionaries

### 문제

두 dictionary가 공통으로 가지는 것(같은 키, 같은 값 등)을 찾고 싶다.

### 해법

`keys()`나 `items()` 메서드를 사용하여 일반적인 set 연산을 수행한다.

```python
a = {
    'x' : 1,
    'y' : 2,
    'z' : 3
}
b = {
    'w' : 10,
    'x' : 11,
    'y' : 2
}

# 공통 키 찾기
a.keys() & b.keys()  # { 'x', 'y' }

# a에 있지만 b에 없는 키
a.keys() - b.keys()  # { 'z' }

# 공통 (key,value) 쌍
a.items() & b.items()  # { ('y', 2) }
```

Dictionary comprehension으로 특정 키를 제거한 새 dictionary를 만들 수 있다:

```python
c = {key:a[key] for key in a.keys() - {'z', 'w'}}
# c is {'x': 1, 'y': 2}
```

### 논의

Dictionary의 `keys()` 메서드는 union, intersection, difference 같은 일반적인 set 연산을 지원하는 keys-view 객체를 반환한다.

`items()` 메서드는 `(key, value)` 쌍으로 구성된 items-view 객체를 반환하며, 유사한 set 연산을 지원한다.

`values()` 메서드는 값이 고유하다는 보장이 없어 이러한 set 연산을 지원하지 않는다.

## 1.10. Removing Duplicates from a Sequence while Maintaining Order

### 문제

Sequence에서 중복 값을 제거하되, 나머지 항목의 순서를 유지하고 싶다.

### 해법

Sequence의 값이 hashable하면 set과 generator를 사용하여 쉽게 해결할 수 있다.

```python
def dedupe(items):
    seen = set()
    for item in items:
        if item not in seen:
            yield item
            seen.add(item)

>>> a = [1, 5, 2, 1, 9, 1, 5, 10]
>>> list(dedupe(a))
[1, 5, 2, 9, 10]
```

Unhashable type(예: dict)의 중복을 제거하려면 `key` 매개변수를 추가한다:

```python
def dedupe(items, key=None):
    seen = set()
    for item in items:
        val = item if key is None else key(item)
        if val not in seen:
            yield item
            seen.add(val)

>>> a = [ {'x':1, 'y':2}, {'x':1, 'y':3}, {'x':1, 'y':2}, {'x':2, 'y':4}]
>>> list(dedupe(a, key=lambda d: (d['x'],d['y'])))
[{'x': 1, 'y': 2}, {'x': 1, 'y': 3}, {'x': 2, 'y': 4}]
>>> list(dedupe(a, key=lambda d: d['x']))
[{'x': 1, 'y': 2}, {'x': 2, 'y': 4}]
```

### 논의

단순히 중복을 제거하려면 set을 만들면 되지만, 순서가 보존되지 않는다.

```python
>>> set(a)
{1, 2, 10, 5, 9}
```

Generator function을 사용하면 list 처리에만 국한되지 않고 범용적으로 사용할 수 있다.

## 1.11. Naming a Slice

### 문제

Hardcoded slice index로 인해 코드가 읽기 어려워졌다.

### 해법

Slice에 이름을 부여한다.

```python
###### 0123456789012345678901234567890123456789012345678901234567890'
record = '....................100 .......513.25 ..........'

# Before
cost = int(record[20:32]) * float(record[40:48])

# After - much clearer
SHARES = slice(20, 32)
PRICE = slice(40, 48)
cost = int(record[SHARES]) * float(record[PRICE])
```

### 논의

내장 `slice()`는 slice가 허용되는 모든 곳에서 사용할 수 있는 slice 객체를 생성한다.

```python
>>> items = [0, 1, 2, 3, 4, 5, 6]
>>> a = slice(2, 4)
>>> items[a]
[2, 3]
>>> items[a] = [10, 11]
>>> items
[0, 1, 10, 11, 4, 5, 6]
```

Slice instance는 `s.start`, `s.stop`, `s.step` 속성을 가진다.

```python
>>> a = slice(10, 50, 2)
>>> a.start
10
>>> a.stop
50
>>> a.step
2
```

## 1.12. Determining the Most Frequently Occurring Items in a Sequence

### 문제

Sequence에서 가장 자주 나타나는 항목을 찾고 싶다.

### 해법

`collections.Counter` 클래스를 사용한다.

```python
words = [
    'look', 'into', 'my', 'eyes', 'look', 'into', 'my', 'eyes',
    'the', 'eyes', 'the', 'eyes', 'the', 'eyes', 'not', 'around', 'the',
    'eyes', "don't", 'look', 'around', 'the', 'eyes', 'look', 'into',
    'my', 'eyes', "you're", 'under'
]

from collections import Counter
word_counts = Counter(words)
top_three = word_counts.most_common(3)
print(top_three)
# [('eyes', 8), ('the', 5), ('look', 4)]
```

### 논의

`Counter`는 내부적으로 항목을 발생 횟수에 매핑하는 dictionary다.

```python
>>> word_counts['eyes']
8
```

Count를 수동으로 증가시키거나 `update()` 메서드를 사용할 수 있다:

```python
>>> morewords = ['why','are','you','not','looking','in','my','eyes']
>>> word_counts.update(morewords)
```

`Counter` 인스턴스는 수학 연산으로 쉽게 결합할 수 있다:

```python
>>> a = Counter(words)
>>> b = Counter(morewords)
>>> c = a + b  # Combine counts
>>> d = a - b  # Subtract counts
```

## 1.13. Sorting a List of Dictionaries by a Common Key

### 문제

Dictionary의 리스트를 하나 이상의 dictionary 값에 따라 정렬하고 싶다.

### 해법

`operator` 모듈의 `itemgetter` 함수를 사용한다.

```python
rows = [
    {'fname': 'Brian', 'lname': 'Jones', 'uid': 1003},
    {'fname': 'David', 'lname': 'Beazley', 'uid': 1002},
    {'fname': 'John', 'lname': 'Cleese', 'uid': 1001},
    {'fname': 'Big', 'lname': 'Jones', 'uid': 1004}
]

from operator import itemgetter
rows_by_fname = sorted(rows, key=itemgetter('fname'))
rows_by_uid = sorted(rows, key=itemgetter('uid'))
```

`itemgetter()`는 여러 키를 받을 수도 있다:

```python
rows_by_lfname = sorted(rows, key=itemgetter('lname','fname'))
```

### 논의

`itemgetter()`는 lambda 표현식으로 대체될 수 있지만, 일반적으로 `itemgetter()`가 더 빠르다.

```python
rows_by_fname = sorted(rows, key=lambda r: r['fname'])
```

이 기법은 `min()`과 `max()` 함수에도 적용할 수 있다:

```python
>>> min(rows, key=itemgetter('uid'))
{'fname': 'John', 'lname': 'Cleese', 'uid': 1001}
```

## 1.14. Sorting Objects Without Native Comparison Support

### 문제

같은 클래스의 객체들을 정렬하고 싶지만, 비교 연산을 기본적으로 지원하지 않는다.

### 해법

`sorted()` 함수의 `key` 매개변수에 callable을 전달한다.

```python
>>> class User:
...     def __init__(self, user_id):
...         self.user_id = user_id
...     def __repr__(self):
...         return 'User({})'.format(self.user_id)

>>> users = [User(23), User(3), User(99)]
>>> sorted(users, key=lambda u: u.user_id)
[User(3), User(23), User(99)]
```

Lambda 대신 `operator.attrgetter()`를 사용할 수 있다:

```python
>>> from operator import attrgetter
>>> sorted(users, key=attrgetter('user_id'))
[User(3), User(23), User(99)]
```

### 논의

`attrgetter()`는 종종 약간 더 빠르며, 여러 필드를 동시에 추출할 수 있다:

```python
by_name = sorted(users, key=attrgetter('last_name', 'first_name'))
```

이 기법은 `min()`과 `max()`에도 적용된다:

```python
>>> min(users, key=attrgetter('user_id'))
User(3)
```

## 1.15. Grouping Records Together Based on a Field

### 문제

Dictionary나 인스턴스의 sequence가 있고, 특정 필드 값에 따라 그룹으로 데이터를 반복하고 싶다.

### 해법

`itertools.groupby()` 함수를 사용한다.

```python
rows = [
    {'address': '5412 N CLARK', 'date': '07/01/2012'},
    {'address': '5148 N CLARK', 'date': '07/04/2012'},
    {'address': '5800 E 58TH', 'date': '07/02/2012'},
    {'address': '2122 N CLARK', 'date': '07/03/2012'},
    {'address': '5645 N RAVENSWOOD', 'date': '07/02/2012'},
    {'address': '1060 W ADDISON', 'date': '07/02/2012'},
    {'address': '4801 N BROADWAY', 'date': '07/01/2012'},
    {'address': '1039 W GRANVILLE', 'date': '07/04/2012'},
]

from operator import itemgetter
from itertools import groupby

# 먼저 원하는 필드로 정렬
rows.sort(key=itemgetter('date'))

# 그룹으로 반복
for date, items in groupby(rows, key=itemgetter('date')):
    print(date)
    for i in items:
        print(' ', i)
```

### 논의

`groupby()` 함수는 sequence를 스캔하고 연속된 동일한 값의 "run"을 찾는다. 각 반복마다 값과 함께 해당 그룹의 모든 항목을 생성하는 iterator를 반환한다.

중요한 전제 단계는 관심 필드로 데이터를 정렬하는 것이다. `groupby()`는 연속된 항목만 검사하므로, 먼저 정렬하지 않으면 원하는 대로 레코드가 그룹화되지 않는다.

Random access가 필요하다면 `defaultdict`를 사용하여 multidict를 구축하는 것이 더 나을 수 있다:

```python
from collections import defaultdict
rows_by_date = defaultdict(list)
for row in rows:
    rows_by_date[row['date']].append(row)
```

## 1.16. Filtering Sequence Elements

### 문제

Sequence 내부의 데이터에서 값을 추출하거나 특정 기준으로 sequence를 줄여야 한다.

### 해법

List comprehension이 가장 쉬운 방법이다.

```python
>>> mylist = [1, 4, -5, 10, -7, 2, 3, -1]
>>> [n for n in mylist if n > 0]
[1, 4, 10, 2, 3]
>>> [n for n in mylist if n < 0]
[-5, -7, -1]
```

입력이 크면 generator expression을 사용하여 반복적으로 필터링된 값을 생성할 수 있다:

```python
>>> pos = (n for n in mylist if n > 0)
>>> for x in pos:
...     print(x)
1
4
10
2
3
```

필터링 기준이 복잡하면 함수로 만들고 내장 `filter()` 함수를 사용한다:

```python
values = ['1', '2', '-3', '-', '4', 'N/A', '5']

def is_int(val):
    try:
        x = int(val)
        return True
    except ValueError:
        return False

ivals = list(filter(is_int, values))
# ['1', '2', '-3', '4', '5']
```

### 논의

List comprehension과 generator expression은 동시에 데이터를 변환할 수도 있다:

```python
>>> import math
>>> [math.sqrt(n) for n in mylist if n > 0]
[1.0, 2.0, 3.1622776601683795, 1.4142135623730951, 1.7320508075688772]
```

기준을 만족하지 않는 값을 새 값으로 대체할 수도 있다:

```python
>>> clip_neg = [n if n > 0 else 0 for n in mylist]
>>> clip_neg
[1, 4, 0, 10, 0, 2, 3, 0]
```

`itertools.compress()`는 iterable과 Boolean selector sequence를 입력으로 받아 필터링에 유용하다:

```python
>>> from itertools import compress
>>> addresses = ['5412 N CLARK', '5148 N CLARK', '5800 E 58TH', ...]
>>> counts = [0, 3, 10, 4, 1, 7, 6, 1]
>>> more5 = [n > 5 for n in counts]
>>> list(compress(addresses, more5))
['5800 E 58TH', '1060 W ADDISON', '4801 N BROADWAY']
```

## 1.17. Extracting a Subset of a Dictionary

### 문제

다른 dictionary의 부분집합인 dictionary를 만들고 싶다.

### 해법

Dictionary comprehension을 사용한다.

```python
prices = {
    'ACME': 45.23,
    'AAPL': 612.78,
    'IBM': 205.55,
    'HPQ': 37.20,
    'FB': 10.75
}

# 200 이상의 모든 가격으로 dictionary 만들기
p1 = {key:value for key, value in prices.items() if value > 200}

# 기술주 dictionary 만들기
tech_names = {'AAPL', 'IBM', 'HPQ', 'MSFT'}
p2 = {key:value for key,value in prices.items() if key in tech_names}
```

### 논의

Dictionary comprehension 솔루션이 더 명확하고 실제로 더 빠르게 실행된다.

## 1.18. Mapping Names to Sequence Elements

### 문제

위치로 list나 tuple 요소에 접근하는 코드가 있지만, 이름으로 접근하고 싶다.

### 해법

`collections.namedtuple()`을 사용한다. 이는 일반 tuple 객체에 최소한의 오버헤드를 추가하면서 이점을 제공한다.

```python
>>> from collections import namedtuple
>>> Subscriber = namedtuple('Subscriber', ['addr', 'joined'])
>>> sub = Subscriber('jonesy@example.com', '2012-10-19')
>>> sub
Subscriber(addr='jonesy@example.com', joined='2012-10-19')
>>> sub.addr
'jonesy@example.com'
>>> sub.joined
'2012-10-19'
```

`namedtuple` 인스턴스는 일반 class 인스턴스처럼 보이지만 tuple과 교환 가능하며 indexing과 unpacking 같은 일반적인 tuple 연산을 지원한다.

```python
>>> len(sub)
2
>>> addr, joined = sub
```

### 논의

Named tuple의 주요 사용 사례는 조작하는 요소의 위치로부터 코드를 분리하는 것이다.

```python
# Before - position-dependent
def compute_cost(records):
    total = 0.0
    for rec in records:
        total += rec[1] * rec[2]
    return total

# After - using namedtuple
from collections import namedtuple
Stock = namedtuple('Stock', ['name', 'shares', 'price'])

def compute_cost(records):
    total = 0.0
    for rec in records:
        s = Stock(*rec)
        total += s.shares * s.price
    return total
```

`namedtuple`은 immutable이다. 속성을 변경하려면 `_replace()` 메서드를 사용한다:

```python
>>> s = Stock('ACME', 100, 123.45)
>>> s = s._replace(shares=75)
>>> s
Stock(name='ACME', shares=75, price=123.45)
```

## 1.19. Transforming and Reducing Data at the Same Time

### 문제

Reduction 함수(예: `sum()`, `min()`, `max()`)를 실행해야 하지만, 먼저 데이터를 변환하거나 필터링해야 한다.

### 해법

Generator-expression을 인자로 사용한다.

```python
nums = [1, 2, 3, 4, 5]
s = sum(x * x for x in nums)

# .py 파일이 디렉토리에 존재하는지 확인
import os
files = os.listdir('dirname')
if any(name.endswith('.py') for name in files):
    print('There be python!')

# Tuple을 CSV로 출력
s = ('ACME', 50, 123.45)
print(','.join(str(x) for x in s))

# 데이터 구조의 필드 전체에서 data reduction
portfolio = [
    {'name':'GOOG', 'shares': 50},
    {'name':'YHOO', 'shares': 75},
    {'name':'AOL', 'shares': 20},
    {'name':'SCOX', 'shares': 65}
]
min_shares = min(s['shares'] for s in portfolio)
```

### 논의

함수의 단일 인자로 제공될 때 generator expression은 중복 괄호가 필요 없다:

```python
s = sum((x * x for x in nums))  # OK
s = sum(x * x for x in nums)    # 더 우아한 구문
```

Generator 인자를 사용하는 것이 임시 list를 먼저 만드는 것보다 더 효율적이고 우아하다:

```python
# 작동하지만 추가 리스트를 생성함
s = sum([x * x for x in nums])
```

`min()`과 `max()` 같은 reduction 함수는 `key` 인자를 받는다:

```python
# Original: 20 반환
min_shares = min(s['shares'] for s in portfolio)

# Alternative: {'name': 'AOL', 'shares': 20} 반환
min_shares = min(portfolio, key=lambda s: s['shares'])
```

## 1.20. Combining Multiple Mappings into a Single Mapping

### 문제

여러 dictionary나 mapping을 논리적으로 단일 mapping으로 결합하여 값을 찾거나 키의 존재를 확인하고 싶다.

### 해법

`collections` 모듈의 `ChainMap` 클래스를 사용한다.

```python
a = {'x': 1, 'z': 3}
b = {'y': 2, 'z': 4}

from collections import ChainMap
c = ChainMap(a, b)
print(c['x'])  # 1 (from a)
print(c['y'])  # 2 (from b)
print(c['z'])  # 3 (from a)
```

### 논의

`ChainMap`은 여러 mapping을 받아 논리적으로 하나로 보이게 만든다. 하지만 실제로 병합되지는 않는다. 대신 underlying mapping의 리스트를 유지하고 공통 dictionary 연산을
재정의하여 리스트를 스캔한다.

```python
>>> len(c)
3
>>> list(c.keys())
['x', 'y', 'z']
>>> list(c.values())
[1, 2, 3]
```

중복 키가 있으면 첫 번째 mapping의 값이 사용된다.

Mapping을 변경하는 연산은 항상 첫 번째 mapping에 영향을 준다:

```python
>>> c['z'] = 10
>>> c['w'] = 40
>>> a
{'w': 40, 'z': 10}
```

`ChainMap`은 프로그래밍 언어의 변수(globals, locals 등)처럼 scoped value와 함께 작업할 때 특히 유용하다:

```python
>>> values = ChainMap()
>>> values['x'] = 1
>>> values = values.new_child()
>>> values['x'] = 2
>>> values = values.new_child()
>>> values['x'] = 3
>>> values
ChainMap({'x': 3}, {'x': 2}, {'x': 1})
>>> values = values.parents
>>> values['x']
2
```

대안으로 `update()` 메서드로 dictionary를 병합할 수 있지만, 완전히 별도의 dictionary 객체를 만들어야 하며, 원본 dictionary가 변경되어도 반영되지 않는다. `ChainMap`은
원본 dictionary를 사용하므로 이런 문제가 없다.

```python
>>> a = {'x': 1, 'z': 3}
>>> b = {'y': 2, 'z': 4}
>>> merged = ChainMap(a, b)
>>> merged['x']
1
>>> a['x'] = 42
>>> merged['x']  # 변경사항이 반영됨
42
```